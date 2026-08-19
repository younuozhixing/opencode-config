# Engineering rules

Companion to [`CLAUDE.md`](CLAUDE.md). `CLAUDE.md` is *how* you behave
when you write code; this file is *what to honour* across any
Unomove codebase — current or future.

Pure rules. No project, repo, file, or hardware mentions. If a rule
references a domain (a vehicle bus, a GUI framework, a key-value
config store, an embedded target), it's written so the rule
transfers to the next project in that domain without edits.

The deliberate exceptions are the org's **positive conventions**,
named on purpose because every project shares them: the
**uno-namespace** for user-visible identifiers ([§1](#naming)), the
shared library **`unolib`** as the home for reusable, Rust-first code
([§9](#rust-vs-python), [§28](#reuse-critical-path)), and the
**Song-minimalist house visual language** every GUI adopts
([§8.28](ui-rules.md#house-design-language)).

If you're touching a specific codebase and want the project-local
notes (where a service config lives, what the bring-up command is),
those go in that project's own `CLAUDE.local.md` or equivalent.
This file stays neutral.

**Each section opens with a `> When this fires … / Do …` header** so
it reads as a self-contained *skill*: land on it from the
[`CLAUDE.md`](CLAUDE.md) trigger index, see in one glance whether it's
yours and the one move it asks for, then read the depth only if it is.
(Anti-patterns already work this way — symptom → looks-like →
actually-is → fix.) Sub-rules inherit their parent's *when*; the gate
([`check.py`](check.py)) requires every section to carry the header.

## Contents

The 28 sections are grouped into **eight parts** by what kind of
work they govern. Sections keep stable numbers (and stable slug
anchors) — the parts are a reading map over them, not a
renumbering. Jump by task via the trigger index in
[`CLAUDE.md`](CLAUDE.md), scan everything in one page via
[`quickref.md`](quickref.md), or read a whole theme via a part
below.

**A — Interfaces & data contracts** *(what crosses a boundary and must stay stable)*
- 1. [Naming](#naming)
- 2. [Units](#units)
- 4. [Schemas](#schemas)
- 5. [Messaging hot path](#messaging-hot-path)

**B — Correctness & safety** *(getting behaviour right, especially where it's irreversible)*
- 3. [Safety-critical changes](#safety-critical)
- 7. [Silent-default traps](#silent-defaults)
- 9. [Choosing Rust vs. Python](#rust-vs-python)
- 28. [Reuse the critical path](#reuse-critical-path)
- 29. [Control flow: dispatch over deep conditionals](#control-flow-dispatch)

**C — Runtime: processes, streams, logs** *(what's alive and how it behaves under load)*
- 6. [Process supervision](#process-supervision)
- 10. [Per-stream isolation + shared probes](#per-stream-isolation)
- 16. [Logging discipline](delivery-rules.md#logging)

**D — User interfaces**
- 8. [UI architecture](ui-rules.md#ui-architecture) — *in its own file, [`ui-rules.md`](ui-rules.md)*

**E — Build, deploy & fleet** *(getting code onto many targets, identically)*
- 11. [Cross-compile](delivery-rules.md#cross-compile)
- 12. [Syncing source to an embedded target](delivery-rules.md#embedded-sync)
- 13. [Source tree vs. deployed artifact](delivery-rules.md#source-vs-deployed)
- 14. [Bundled configuration and shipping defaults](delivery-rules.md#bundled-config)
- 21. [Python environment management](delivery-rules.md#python-env)
- 27. [Fixing a build systematically](delivery-rules.md#build-fix)

**F — Diagnose & verify** *(proving a change works, finding why it doesn't)*
- 15. [Diagnostics](delivery-rules.md#diagnostics)
- 17. [Validation surfaces](delivery-rules.md#validation-surfaces)
- 18. [Tests / verification](delivery-rules.md#tests)

**G — Working safely in the repo** *(the moves around the code, not the code)*
- 19. [Reversibility](delivery-rules.md#reversibility)
- 20. [Repo hygiene](delivery-rules.md#repo-hygiene)
- 22. [When you add a new daemon, service, or topic](delivery-rules.md#new-service-checklist)
- 24. [When in doubt](delivery-rules.md#when-in-doubt)

**H — Integration & external surfaces** *(where the system meets the outside world — the network edge, services you call, the users you serve)*
- 23. [Network topology](delivery-rules.md#network-topology)
- 25. [Integrating with an external service / API](delivery-rules.md#external-integration)
- 26. [Handling money / privileged actions](delivery-rules.md#trust-boundary)

[Appendix: How to add to this rule book](#how-to-add)

**Why numbers are stable, not contiguous-per-part.** Cross-references
throughout (here and in the other files) cite sections by number
*and* slug (`[§8.19](ui-rules.md#wire-state-mirror)`). The slug keeps the link
alive; the number must keep meaning what it says. So sections are
never renumbered — new ones append, and the parts above regroup the
fixed numbers thematically. A part's section numbers are therefore
not a contiguous run, and that's deliberate: stable references beat
tidy ranges.

---

<a id="naming"></a>
## 1. Naming

> **When this fires** — adding or renaming any user-visible string (UI label, log, error, CLI name, config key, public module / crate).
>
> **Do** — put it in the **uno-namespace**; external addresses (URLs, CI, third-party) are exempt.

User-visible strings (UI labels, log lines, error messages, README
prose, CLI command names, env var names, config keys, service
names, public language-level module / package / crate paths) live
in the **uno-namespace**:

> `uno*`, `unomove`, `unocoding`.

That's the positive rule. Specific corollaries:

- A library or project we maintain has a uno-prefixed identity.
  This applies to crate names, package names, Python modules, and
  service-bus topic names alike.
- A toggle, mode, or feature flag carries a name that doesn't
  collide with adjacent vocabulary. If your build system already
  has the word "debug" loaded (debug build, debug logging, debug
  symbols), don't reuse it for an unrelated user-mode toggle —
  pick a different word.
- The *absence* of cross-stack compatibility surfaces is sometimes
  a deliberate design property, not a debt. Don't add wire-compat
  shims, identity prefixes, or magic numbers from another stack
  "for less divergence." The lack of those surfaces is what we got
  out of.
- A non-uno identifier that has to stay for byte-level
  compatibility (a vendored schema field, a third-party magic
  number) lives in a low-level constants file with an inline
  comment explaining why it can't be renamed.

If you find yourself typing a non-uno name into a string the user
will see, stop. It's a leak; flag it.

### External names are load-bearing — exempt from a rename sweep

When you run a namespace/rebrand rename across a tree, **a name
that resolves against an external system is not yours to rename.**
Renaming it doesn't re-brand anything — it 404s. Exempt, with a
comment saying why:

- **URLs and endpoints** to systems you don't own (upstream repo
  URLs, a cloud bucket / blob path, a third-party API host). The
  remote side still expects the old name; rename → broken link.
- **CI/infra identifiers** that name external resources — a build
  cluster, a container registry path, a storage account. These
  are addresses, not labels.
- **Third-party / vendored subtrees** (an upstream library
  checked into the tree). Not your namespace; renaming forks it
  from upstream and breaks the next sync.
- **Attribution / provenance comments** ("copied from upstream X")
  — renaming them launders the provenance, which is the opposite
  of what they're for.

The test for "is this exempt": **does the string resolve against
something outside our control?** If yes, it's an address — leave
it. If it's purely our own user-facing identity, it follows the
uno-namespace rule above. A first-party `import`/symbol *is* ours
— rename it; the URL three lines down probably isn't.

<a id="rewrite-clean-break"></a>
### A ground-up rewrite is a clean break — its own namespace and structure, not the legacy's

A from-scratch rewrite *replaces* a legacy system; it must not quietly
become a **fork of it**. Because a rewrite re-assembles known pieces, the
legacy's **names, module layout, identifiers, and identity keep creeping
back in** — a type named after the old class, a directory tree mirroring
the old one, the upstream project's name in a symbol or a log tag — until
the "new" system is the old one in a new coat and every future change
fights the legacy's shape. "Less divergence" is the smell; divergence was
the point.

- **The new namespace and conventions run *all the way through*** — not
  just the user-visible strings above, but internal modules, types,
  crate / package layout, CLI verbs, log tags. Don't carry the legacy's
  identifiers *inward* "to port faster"; that **is** the drift.
- **The tell is the carried *names*, not the comments.** The pervasive
  form is the legacy's **identifiers recreated one-for-one** — its
  daemon / process names (a whole `*d` set), topic / field names, module
  and type names, config paths — so the new tree reads as the old one
  with the imports swapped. Those are the drift; rename them to the new
  idiom. A **provenance comment** ("a faithful port of `<legacy>`'s
  `LongControl`") is the *opposite* and is **exempt** ([§1 external names
  are load-bearing](#naming)) — it records lineage, it doesn't make the
  new system a fork. Audit the **identifiers**; keep the attribution.
- **Depend on the legacy only at one deliberate, narrow seam.** Where
  interop is genuinely required — a shared wire so an existing consumer
  keeps working — keep it to a single explicit boundary and **share the
  schema as data** (lineage-guarded, [§28.2](#cross-language-lib)), never
  the legacy's code, types, or build. *Wire-compatible is not
  code-coupled*: "we need compat" must not become "import the old stack."
- **Re-state defaults from your own source of truth.** A fork silently
  inherits the upstream's tables, paths, and magic numbers
  ([§2.3](#inherited-hardware-constants)); a rewrite restates them in its
  own idiom.
- **Record that it's a clean rewrite** ([§18.3](delivery-rules.md#decision-record)):
  "independent of <legacy>, interop only at <seam>." Then drift back
  toward the legacy is a **regression against the rewrite's whole reason
  to exist** that a reviewer can name — not an innocent convenience.

The generalisable rule: **a ground-up rewrite is a clean break, not a
fork — carry the new system's namespace, structure, and identity all the
way through; depend on the legacy only at one deliberate, narrow interop
seam (share the schema as data, never the code); and treat any drift back
into the legacy's names, layout, or identity as a regression.** See
anti-pattern #104.

---

<a id="units"></a>
## 2. Units

> **When this fires** — a message / API field carries a physical quantity, a timestamp that gets correlated, or pixels tied to a resolution.
>
> **Do** — name the unit explicitly and keep it SI; one shared clock / epoch / unit for correlated streams; pin intrinsics to one resolution; re-derive inherited hardware constants.

Every numeric field in a message, log, or API surface carries SI
units unless the name says otherwise:

- Speed → **m/s**.
- Angle → **rad**.
- Time → **s** (or **ns** for monotonic timestamps).
- Distance → **m**. Mass → **kg**. Power → **W**. Voltage → **V**.

If a non-SI unit is genuinely needed (vendor sensor reports mph),
encode the unit in the name: `*_mph`, `*_deg`, `*_kmh`. The unit
is **never** implicit.

Field names must be unambiguous in the context of their message.
`speed` alone is wrong unless every reader can be sure it's m/s;
`speed_mps` or `ground_speed` makes intent explicit.

Values should be easy to plot and human-readable with minimal
parsing. Use real enums, not bit-packed ints. Booleans, not int
flags.

<a id="timestamp-contract"></a>
### 2.1 A timestamp is a contract: one shared clock, one epoch, one unit

When two streams are ever **correlated by timestamp** — a consumer
that *synchronizes* frames from several producers (match a road and
a wide camera within 10 ms), a log merge, a sensor-fusion window —
the timestamps must come from **one shared clock, with one epoch
and one unit.** A per-producer clock silently desynchronizes them:

- **One epoch.** If each producer stamps from its own zero — each
  GStreamer pipeline's `base_time`, each process's start — the
  values are not comparable across producers even though they wear
  the same field name. We shipped this: each camera was its own
  pipeline, so `GST_BUFFER_PTS` (`clock − per-pipeline base_time`)
  put two front cameras ~25 s apart. The consumer matched within a
  10 ms tolerance, so the gap could never close — it spun at ~42 %
  CPU and **never produced output.** Stamp at the common boundary
  with a shared monotonic clock (`CLOCK_BOOTTIME` /
  `nanos_since_boot()`), not the per-pipeline PTS.
- **One unit.** A timestamp is a §2 number — ns or s, never
  ambiguous. The same regression also divided `pts_ns / 1000` (µs
  where ns was required): an off-by-1000 that desyncs every
  consumer. Name it (`*_ns`, `*_mono_ns`) and keep it ns
  end-to-end.
- **The failure looks like the consumer, not the clock.** "The
  model is alive but publishes 0 Hz" / "frames out of sync" spam
  reads as a broken consumer; it's the timestamp contract upstream.
  When a *correlating* consumer stalls with no output, suspect the
  clock domain before the consumer
  ([looks-healthy ≠ working, #2](anti-patterns.md#recurring-shapes)).

The generalisable rule: **timestamps that are ever compared must
share one clock, one epoch, and one declared unit — a per-source
clock or a unit slip silently desynchronizes correlated streams and
stalls the consumer that joins them, while looking like that
consumer's bug.** See anti-pattern #48.

<a id="resolution-contract"></a>
### 2.2 A geometric calibration is tied to its resolution — pin one across the chain, or scale it

Camera **intrinsics** (focal `fx,fy`, principal point `cx,cy`), a
homography, lens-distortion coefficients — any geometric
calibration — are computed **in pixels at a specific image
resolution**, and are valid *only* at that resolution. Apply a
1920-wide intrinsic matrix to a 1280-wide stream and every term is
off by the scale factor — a perfectly centred `cx,cy` lands ~33 %
off-centre, the focal is wrong — **silently**, as skewed geometry:
the model's warp, the lane overlay, the projection all drift, and
nothing throws.

Hold one **resolution contract** across the whole chain:

- **Pin a single resolution — the one the *consumer* (the model)
  actually ingests** — and make **capture, calibration, the runtime
  intrinsics, and any overlay** use it. Calibrate at the *served*
  resolution; don't calibrate at full sensor res and feed a
  downscaled stream.
- **If a stage must run at a different resolution, scale the
  intrinsics** (`fx,cx ×= w_t/w_c`, `fy,cy ×= h_t/h_c`) — never pass
  the raw matrix across a resolution change.
- **The resolution is part of the calibration artifact.** Store it
  next to `fx,fy,cx,cy` and check it on load — the same "a number is
  meaningless without the frame it's in" idea as
  [§2.1](#timestamp-contract) (pixels instead of time), and what a
  safety-critical calibration gate ([§3.5](#validate-safety-input))
  must verify.

The generalisable rule: **a geometric calibration is valid only at
the resolution it was computed in — pin one resolution across
capture → calibrate → consume → overlay, scale the intrinsics for
any stage that differs, and store the resolution with the
calibration; a resolution mismatch silently corrupts geometry, it
doesn't error.** See anti-pattern #66.

<a id="inherited-hardware-constants"></a>
### 2.3 A forked or ported stack inherits the upstream's hardware constants — re-derive them for *your* hardware

When you fork a stack, or port it onto hardware that differs from
what its authors targeted, every **hardware constant baked into the
source describes their hardware, not yours**: the camera model and
its field of view, intrinsics and mounting geometry, sensor scaling
and bias, the device/sensor table the code selects from, the shipped
calibration defaults. None of it crashes on your hardware — the
numbers load, the model runs — but the device they encode is the
wrong one, so perception and geometry come out **quietly wrong**.

We shipped this: a ported perception stack kept the upstream's
default camera table — a different sensor with a different field of
view, a principal point and focal length stored at a resolution the
actual stream never runs at. The intrinsics loaded without error and
the model produced output; the geometry was skewed (principal point
tens of percent off the true centre, focal anisotropic) because the
constants were a *reference* unit's, not the installed one.

- **Treat every inherited hardware constant as a placeholder until
  measured on the actual unit.** Camera intrinsics, FOV,
  extrinsics / mounting, sensor ranges — re-derive them by
  calibrating / measuring *this* device; don't trust the value that
  came with the fork.
- **An inherited default that doesn't crash is not validated** — it's
  the [silent-default trap](#silent-defaults) wearing a hardware
  constant. "It loaded and the model ran" says nothing about whether
  the numbers describe *your* camera.
- **Pin each constant to the device it came from** so a mismatch is
  detectable: store the sensor / resolution / unit alongside the
  value (as in [§2.2](#resolution-contract)) and check it against the
  hardware actually present — the same gate a safety-critical
  calibration input gets ([§3.5](#validate-safety-input)).

The generalisable rule: **a forked or ported stack's hardware
constants — camera intrinsics / FOV, sensor model, mounting, device
tables, calibration defaults — are the *upstream's*, valid only for
the upstream's hardware; on a different unit they load fine and
silently encode the wrong device. Re-derive each from the actual
unit and pin it to that unit; never trust an inherited hardware
default that merely doesn't error.** See anti-pattern #70.

---

<a id="safety-critical"></a>
## 3. Safety-critical changes

> **When this fires** — the code can reach an actuator with no hardware backstop (steer / throttle / brake / gripper / valve, engage), or a secondary control surface / takeover feed.
>
> **Do** — name the path and one failure mode, **ask before merging**, and write *less* code on the path — not more.

A change is safety-critical when the software under edit has a
direct authority over a physical actuator (steering, throttle,
brake, gripper, manipulator, valve) **without** a hardware
backstop between it and the actuator. "There is no second
authority" is the test.

For such changes:

1. State explicitly which actuator the change can reach.
2. Name at least one failure mode you considered.
3. Ask before merging — even if the user previously approved an
   unrelated change in the same project. Approval is per-path, not
   blanket.
4. Do not widen actuator limits, accel / jerk caps, or shorten
   safety timeouts on the strength of simulation alone.
5. Load-bearing handshakes between subsystems (engage, prepare,
   safety-state transitions) are not refactor targets. Don't
   collapse them into a generic state machine without verifying on
   real hardware.

When in doubt, write *less* code on the safety-critical path, not
more.

### 3.1 Secondary control surfaces default to view-only

A web GUI, a mobile companion, a "remote-of-the-remote", any
secondary surface that can reach the actuator — **defaults to
view-only**. Sending a control command requires the operator to
explicitly opt in (a "TAKE CONTROL" toggle, a held button, a
confirmed dialog). The server-side endpoint silently drops
control input unless the opt-in is active:

```python
def post_control(req):
    if not state.client_in_control_mode:
        return 204            # no-op, NOT an error
    actuator.apply(parse(req))
```

Why: a forgotten browser tab on the operator's phone must not
move the vehicle when their elbow brushes the screen. The default
mode is observer; control is a deliberate act.

### 3.2 Emergency stop is always on, regardless of mode

The corollary: e-stop / max-brake-latch / kill-switch endpoints
work in **every** mode, including view-only. They don't require
TAKE-CONTROL, don't require auth flows beyond the bare minimum,
and use a minimum hold period so a single tap commits even if the
client disconnects immediately after:

```python
def post_estop(req):
    # No mode check — e-stop ALWAYS works.
    actuator.apply(brake=1.0, throttle=0, steer=0,
                   hold_until=monotonic() + _ESTOP_MIN_HOLD_S)
```

The min-hold matters. Without it, a flaky link that drops one
estop frame and starts re-sending normal control would un-brake
a vehicle that should be stopping.

### 3.3 Safety watchdogs document their threshold in place

Any watchdog that zeroes / fails-safe on input timeout owes its
threshold a comment explaining **which failure modes it catches**
and **why this number**. The watchdog is the last line; future
readers must not "clean up" the constant without knowing what
breaks if they do:

```python
# SAFETY watchdog: zero the axes before the next send if no
# update() arrives in this window. Catches three failure modes
# with one defence:
#   * input subprocess crashes → no callbacks → axes stop refreshing
#   * USB / HID disconnect → backend reconnect loop, seconds gap
#     before it succeeds → operator's last throttle would keep firing
#   * backend silently hangs → no callback, no detection
# Tuned to ~2.5× the consumer's stale-input gate. Smaller would
# clip a brief renegotiation; larger would mean the vehicle keeps
# acting on a stale command after the operator's wire went down.
STALE_INPUT_TIMEOUT_S = 0.5
```

When you tighten a safety threshold, name the failure mode you
were chasing. When you loosen one, name the false-positive you
were eliminating. The constant alone tells future-you nothing.

<a id="control-feed-freshness"></a>
### 3.4 A remote-control / takeover feed is latency-bounded and latest-wins

When a human takes over or teleoperates from a video / sensor feed,
that feed is **part of the control loop** — the operator acts on
what it shows. So it must always show the **latest** frame, not a
smooth-but-late one: **recency beats completeness, fps, and
resolution.** A playback buffer that smooths jitter by queueing
frames is exactly wrong here — every queued frame is reality the
operator is *behind*, and on a control surface that is a safety
hazard, not a cosmetic glitch.

Build the feed to re-converge to "now":

- **Latest-wins, not play-every-frame.** The display reads a
  **single-slot, latest-wins** buffer (show whatever the receive
  loop last wrote), not a FIFO that plays a backlog. A late frame
  is *dropped*, not shown.
- **Bound the latency, not the frame loss.** Cap the receive
  backlog and any decode queue to a small **time** budget (a few
  frames), tuned to a small multiple of the baseline transport
  latency; past it, drop to the newest. On a jittery link that
  means **lower fps, not growing lag** — the right trade for
  control. (One deployment: backlog-drop ~250 ms, decode depth ~2
  frames at 25 fps.)
- **Adapt with a closed loop on an *unmaskable* signal — not static
  presets.** Trade picture for recency at the source: a clear
  lower-res near-live frame beats a crisp stale one. Drive it with
  **AIMD** off the *measured* congestion signal — cut fast the
  instant it rises, probe up slowly when clear, clamped
  floor↔ceiling. But pick a signal **nothing else holds artificially
  flat**: a send-queue depth that a frame-dropper keeps low *masks*
  the stall — the loop reads "clear" and climbs straight into the
  congestion — so key on the true **drop-rate**, and fold in a
  far-side signal the local side can't see (**receiver RTT**: back
  off on high RTT even with an empty send queue).
- **Adapt on the *total* budget and divide it among consumers — never
  one independent loop per consumer.** With one AIMD loop *per stream*,
  each loop probes up into headroom the *others* are also claiming — so
  at low cap they all read "clear," climb together, and **multiply demand
  on the one shared link**: collapse → shed all → repeat (a ~minute
  flap). Run a **single controller on the shared budget** and set each
  consumer's share = budget ÷ active senders, so **adding** a stream
  scales every share *down*, never up, and a restore can't multiply
  demand. Concurrent sessions divide the *same* budget (two viewers were
  half one rig's congestion). Smooth it with a **dead-band** (shed and
  restore thresholds apart, not one line) over an **asymmetric EWMA**
  (fast-down for collapses, slow-up to ride the AIMD ripple) so it
  doesn't hunt at the boundary, and keep the throttle to **one writer**
  ([§5.3](#sole-publisher)). This is the general shape for adaptive
  allocation of *any* shared, capped resource — a connection pool, a
  rate limiter across tenants, GPU memory across models — not just video.
- **Use levers that reconfigure *without* re-initializing hardware
  — never live-re-cap the encoder resolution.** The safe knobs
  shrink **bytes (bitrate) + frames (fps) + stream-count**: bitrate
  is a live `set`, fps a frame-drop, stream-count a shed — none
  re-inits the codec. Changing the **encode resolution live tears
  down and re-opens the hardware encoder**, which under load can
  fail to re-acquire a *shared* HW handle and **wedge a shared
  engine, cascading to unrelated consumers** (one rig: a live-res
  re-cap wedged Tegra host1x → every camera to 0 Hz). Resolution is
  a **fixed startup** value, not a live lever
  ([§9.9](#shared-resource-contention), anti-pattern #65). The
  **path type** (the [best-path tracker, §10.1](#shared-probes))
  decides *how many* streams to send — on a thin / shared relay
  **shed secondary streams to the primary (forward) pair** so each
  survivor has bitrate to claim, re-syncing on path flip. Many feeds
  at full bitrate over a thin shared link all collapse to the
  keyframe floor (§9.9); fewer-but-fresh beats many-but-stale.
- **Enforce the latency bound with a send-buffer hard cap — drop
  even keyframes past it.** A latest-wins *receive* policy isn't
  enough if the *send* side lets a slow link back the buffer up to
  megabytes (seconds of stale keyframes). Cap the outbound buffer;
  past the cap, drop frames — keyframes included — until it drains,
  so a slow relay degrades fps, never latency.
- **Say when it's stale.** If the newest frame is older than the
  freshness bound, mark the tile stale rather than implying live
  ([§8.20 alive-but-silent vs broken](ui-rules.md#overlay-staleness)) — a
  frozen-looking *live* tile is the dangerous case.

**Diagnosing a bad tile: the geometry is the tell.** When a tile comes up
gray / streaked *at the correct size*, or a sibling hangs at "waiting for
stream," the fault is **encoder-side, not the renderer** — a display that
reshapes on the writer's own `w,h` cannot emit a correctly-sized *wrong*
picture, so rule the GUI out by construction and **decode the encode bytes
at the source** with plain ffmpeg. Gray-but-sized = the bitrate floored to
DC-only macroblocks (shrink the encode res by a link-profile **preset via a
camerad restart**, never a live re-cap); stuck = a **missing IDR** (the
keyframe request is wired to the wrong platform API and forces zero IDRs).
See anti-pattern #123.

This is the freshness counterpart to
[§3.1 view-only secondary surfaces](#safety-critical): a takeover
surface that *is* allowed to act must not act on stale input. And
it's the **opposite** of a record / replay / analysis feed, where
completeness wins — so the buffering policy is a property of *what
the feed is for*, chosen deliberately, never a default.

The generalisable rule: **a feed inside a real-time control or
takeover loop is latency-bounded and latest-wins — drop stale
frames to stay near-live, trade fps/resolution for recency, and
surface staleness; never let a smoothing buffer put the operator
behind reality. And its adaptive-quality loop budgets the *total*
shared link and divides it among consumers — never one independent
per-stream loop, which collectively overshoots the shared cap and
flaps.** See anti-patterns #52, #94.

<a id="validate-safety-input"></a>
### 3.5 Validate a safety-critical input before it reaches the consumer — fail safe, and verify it took effect

A value computed or entered upstream — a calibration, a config, a
map — that feeds a safety-critical consumer (the driving model, the
controller) is **untrusted until validated.** A corrupt one must
never reach the consumer; it falls back to a known-good default,
loudly. Three parts, all learned from a corrupt calibration that
slipped a gate and skewed the model's geometry:

**1. A single threshold can't tell legitimate skew from corruption
— gate on *correlated* symptoms.** Real-world data carries
consistent, explainable anomalies (a real ~20 % principal-point
offset that is *symmetric*); corruption stacks *independent* ones
(a big off-centre **and** an impossible aspect ratio **and**
out-of-frame). A lone hard threshold either rejects the
legit-but-skewed input (false reject) or, loosened to admit it,
lets the corrupt one through (false accept) — the corrupt sample
passed a "`fx/fy < 1.5`, off-centre = warn-only" gate on 1.13 aniso
+ 25 % off, each individually mild. Reject when **multiple symptoms
co-occur** (`>15 % off-centre AND fx/fy > 1.10`, or any single
*extreme*); accept a single mild anomaly with a warning. Corruption
is the *combination*, not any one number.

**2. Fail safe to a known-good default, loudly.** On reject, the
consumer uses its built-in default — not the bad value — and the
rejection is surfaced. Never silently accept a value that steers
the model wrong. (Same defence-in-depth as
[§14.4](delivery-rules.md#bundled-config): the input is a *seed* the runtime
validates, not a sealed source.)

And *loudly* outlives the rejection event: **running-on-defaults is
a STATE, not an event** — a log line at reject time evaporates,
while the consumer keeps driving on defaults for hours. The
operator surface must carry a **persistent indicator naming the
real state** ("cameras not calibrated properly — model is on
built-in defaults") for as long as the fallback is active,
re-checked on the target (part 3's gate), cleared only when a valid
input is accepted. A safe fallback the operator can't see is the
silent-default trap ([§7](#silent-defaults)) on a safety path: they
trust geometry/config the consumer isn't using.

**3. Verify it took effect *on the target*, not just that the
deploy ran.** A deploy can "succeed" while the value is rejected at
runtime — so the consumer silently runs on defaults and nobody
knows the calibration never applied. After deploying, re-run the
**target's own acceptance gate at live conditions** (the real
resolution, the real runtime) and report per-item
ACCEPTED/REJECTED. "Deployed" is not "applied"; only the on-target
check proves the consumer is using what you shipped — the
config/calibration analog of [§11.2](delivery-rules.md#freeze-native-deps) ("verify
by importing, not the exit code") and
[§13.1](delivery-rules.md#dev-vs-deployed-layout) ("verify on the production-built
form").

The generalisable rule: **a value feeding a safety-critical
consumer is validated before it can reach it (gate on correlated
symptoms, not one threshold), fails safe to a known-good default
when rejected, and is verified to have actually taken effect on the
target — "deployed" is not "applied," and one mild anomaly is not
corruption.** See anti-pattern #58.

<a id="decision-hysteresis"></a>
### 3.6 A periodically-recomputed decision needs hysteresis — re-validate the current one, don't recompute from scratch

A control or planning loop that **recomputes its output from scratch
every tick** will **flap**. Most non-trivial solvers (a path planner, a
scheduler, a selector, an optimiser) yield a *slightly different* answer
from each slightly-shifted input — so a loop that re-solves every period
publishes a different near-equal solution each time, and everything
downstream **chases a jittering reference**. The output oscillates not
because anything changed meaningfully, but because the decision is being
*re-derived* instead of *held*.

We shipped this: a global path planner recomputed the route every second
regardless of whether the current one was still good; the
Voronoi+smoother pipeline returned a marginally different path from each
new start pose, so the published plan flip-flopped between near-equal
alternatives and the local controller chased the jitter. The fix was to
make the periodic tick **re-validate** the current plan (is every
waypoint still clear and on-grid?) and **recompute only on a genuine
trigger** — the goal changed, a newly-sensed obstacle now crosses the
path, or the downstream is *still* blocked at the end of the period (a
*transient* block it recovers from no longer forces a reroute). Replans
dropped from ~1/s to a handful per run, the path stayed followable, and
hidden-obstacle rerouting still fired.

- **Hold, then re-validate; recompute only on a real trigger.** Keep the
  current decision and each tick ask "is it *still* valid?" — not
  "what's optimal *now*?" Change it when it became invalid, a genuinely
  new input arrived, or a failure **persisted** (not a transient blip).
- **Add hysteresis / a dead-band.** A new candidate must be *meaningfully*
  better (a margin, a debounce, a min-dwell time) before you switch — so
  two near-equal options can't ping-pong. This is the runtime cousin of
  the [decision-record](delivery-rules.md#decision-record) rule (recurring shape #6) and
  the [accelerator-oscillation](#cpu-gpu-decision) rule (§9.7): both say
  *don't re-litigate a settled choice without new evidence* — here the
  "evidence" is a real trigger, not a fresh re-solve.
- **The same shape beyond control loops.** Any periodically re-derived
  selection ping-pongs the same way — a leader/primary election, a
  failover target, an autoscaler replica count, a UI auto-layout, an
  adaptive quality/bitrate level. Re-validate + dead-band, don't
  recompute-and-replace each tick.

The generalisable rule: **a periodically-recomputed decision (a plan,
route, selection, mode, or level) oscillates when it's re-derived from
scratch each tick — near-equal alternatives flap and consumers chase a
jittering reference; instead re-validate the current decision and keep it
while it's still good, recompute only on a real trigger (it became
invalid, a genuine new input, or a sustained — not transient — failure),
and require a margin / dwell-time before switching.** See anti-pattern
#93.

<a id="command-link-safe-stop"></a>
### 3.7 A command / control link safe-stops on staleness — never replay the last command after it drops

A link carrying **live commands to an actuator** — a teleop joystick, a
remote-control channel, a setpoint stream — is part of the control loop.
When it goes silent (operator disconnects, the network drops, the sender
crashes), the consumer must **not keep actuating on the last command it
received.** Holding or replaying a stale command is a **runaway**: a held
throttle keeps the vehicle moving, a frozen steering angle keeps it
turning, long after anyone is steering. "Do what it was last told" is
exactly wrong when the link is dead.

This is the **command-direction** twin of [§3.4](#control-feed-freshness)
(the operator's *display* feed must be fresh) and the safety-critical case
of [§7.6](#stale-pushed-override) (a stale pushed value expires to a safe
default): the feed the operator *watches* must be latest-or-dropped, **and
the command the operator *sends* must expire to a safe state when it stops
arriving.**

We shipped the gap: a joystick consumer actuated on the last
`testJoystick` axes **with no freshness check**, so on link death it
coasted on the operator's last command for up to half a second before
anything noticed. The fix made it bail to the validated idle / coast-to-
stop path once the command aged past a small staleness bound.

- **Check command freshness every tick, bail past a small bound.** Read
  the command's own timestamp (or a sender heartbeat); if it's older than
  a tight threshold (sub-second for a vehicle) drop to a **validated safe
  state** — idle, coast-to-stop, neutral, or hold-brake as the platform
  demands. Recover automatically when fresh commands resume.
- **Key on an *unmaskable* freshness signal.** Not a `connected` flag a
  dead sender leaves `True`, not a buffer that still holds the last value
  — the command's actual arrival time or a heartbeat the sender must keep
  beating ([§3.4](#control-feed-freshness)'s unmaskable-signal rule, on
  the command side).
- **Centralize it in a deterministic watchdog.** One component maps link
  liveness → `Live` / `SafeStop` / `EStop` and the actuator consumer reads
  *that*, not raw axes — so every command path inherits the same
  safe-stop instead of each re-implementing (or forgetting) the check.

The generalisable rule: **a link that actuates must safe-stop when its
command goes stale — check command freshness every tick on an unmaskable
signal (the command's timestamp / a sender heartbeat, never a `connected`
flag), and bail to a validated safe state past a small staleness bound;
never keep replaying the last command after the link drops, because a held
command is a runaway.** See anti-pattern #98.

<a id="proxy-feedback"></a>
### 3.8 Close a control loop on the variable you control, not a proxy that can decouple

A feedback controller is only as good as the measurement it closes on.
The trap is feeding it a **convenient proxy** for the quantity it's
actually meant to control — one that tracks the true variable **most of
the time**, so it looks fine, until a regime where they **decouple**:
then the loop chases the proxy while the plant doesn't follow, and it
**hunts / limit-cycles**.

We shipped this: a longitudinal speed controller closed its PID on a
**motor-rpm-derived** speed instead of true ground speed. Normally rpm
tracks ground speed — but with drivetrain slip / compliance the motor
spun while the body barely moved, so the PID, seeing "speed" swing, drove
current in a ~1 Hz limit cycle (motor 0↔50 A, vehicle near-still) and the
estimator reported impossible accelerations (±12 m/s² on a platform that
commands −4..+1.6). The vehicle was fine; the *feedback signal* was a
proxy that had decoupled, and the loop hunted on it.

- **Sense the controlled variable, not a stand-in.** Close on the real
  quantity (ground speed from wheel odometry / GPS / a fused estimate),
  not the easiest-to-read correlate (motor rpm, an open-loop model
  output). A proxy is acceptable only where it **provably can't decouple**
  within the operating envelope.
- **If a proxy is unavoidable, detect decoupling and disengage.**
  Cross-check it against an independent signal (rpm vs IMU/GPS; encoder vs
  a load cell); when they disagree beyond a bound, treat the feedback as
  invalid — fail to a safe state ([§3.5](#validate-safety-input)) and
  surface it, rather than letting the loop hunt. The plausibility gate
  that catches the limit cycle is the safety net; sensing the true
  variable is the fix.
- **A hunting actuator is the signature.** A near-constant-frequency
  oscillation in the actuator command with the plant *not* responding
  (current/torque pulsing, body steady) is a **feedback** problem, not a
  noisy sensor — look at what the loop closes on before you filter the
  output. (Distinct from [§3.6](#decision-hysteresis) flapping, which is
  recompute-from-scratch, not a decoupled measurement.)

The generalisable rule: **a feedback loop closes on a faithful
measurement of the variable it controls — a proxy / derived signal that
decouples from the true quantity under some regime (wheel/motor slip,
actuator saturation, compliance) makes the loop hunt or limit-cycle;
sense the real variable, or cross-check the proxy against an independent
signal and fail safe when they diverge.** See anti-pattern #100.

---

<a id="schemas"></a>
## 4. Schemas

> **When this fires** — editing a wire or persisted message schema.
>
> **Do** — **append-only** — never renumber or reorder; the wire keys on ordinal, not name.

Public schemas are **append-only**:

- Add fields to existing structs. Don't reorder. Don't renumber.
  Don't delete.
- Don't change a field's type. Add a new field with the new type
  and migrate consumers across.
- Old logs / persisted data must keep round-tripping against the
  new schema.

Don't change the wire format of a published topic without versioning
the topic name (`*_v2`) or coordinating an explicit migration. A
silent change breaks logs and replay forever.

### The wire keys on ordinal, not name

Most schema systems (capnp, protobuf, …) encode each field by its
**numeric tag / ordinal**, not its name. Two consequences that
look surprising until you internalise this:

- **Renaming a field is binary-safe.** `oldName @37` → `newName
  @37` changes the source identifier but not a single wire byte.
  Old recorded logs still decode; a peer on the other end of the
  link that *still uses the old name* still interoperates. So a
  sweeping identifier rename (e.g. a re-brand across the tree) can
  ship without breaking the wire — the rename is a source-level
  change, not a wire change.
- **Renumbering or reordering is fatal**, even with names
  unchanged — because the *ordinal* is the contract. This is why
  the append-only rule above is about field *numbers*, not field
  *names*.

Caveat: a rename is wire-safe but not necessarily *source*-safe
across repos. A separate repo that vendors its own copy of the
schema keeps the old name until it's renamed too; binary interop
holds, but for source-name consistency every copy needs the same
rename. And generated bindings (`gen/`, C++ stubs) and any
pickled/baked artifacts are **build output** — regenerate them on
the target and run the test suite there; a local `py_compile` /
syntax check can't catch a binding/runtime mismatch (§13).

---

<a id="messaging-hot-path"></a>
## 5. Messaging hot path

> **When this fires** — moving bulk data, or polling a queue / socket, on the messaging hot path.
>
> **Do** — send a descriptor on the bus with the bytes in shared memory; **drain all items per wake**, not one per tick.

When two components on the same host need to exchange data, reach
for shared memory first. Don't introduce an intra-host TCP socket,
Unix-domain socket, or HTTP shim unless the SHM path has been
ruled out for a specific reason.

<a id="one-middleware"></a>
**One middleware — `unomsg` — and schemas live in its source.**
`unomsg` is the **only** messaging layer for robots. Never stand up a
second one (no parallel transport, no per-app message *framework*, no
bespoke socket/HTTP bus where unomsg fits) — a second middleware
fractures the wire, the dep graph, and the operator GUI that has to
speak both. A new robot's message **schema** is *not* a new artifact to
publish and version on its own (no standalone `*-msg.git`, no separate
prefix lib): **add the schema into `unomsg`'s own source** — a
feature-gated module the published `unomsg` crate carries
(`unomsg = { …, features = ["slam"] }` → `unomsg::slam::{services, TOPIC_*, …}`).
Every consumer — each daemon *and* the GUI — then gets that schema from
the **single** `unomsg` dependency it already has, so there is one
transport, one dep, and one home for the schema that rides it. A
standalone msg crate is a second source of truth that drifts from the
middleware it serializes for; fold it in instead. *(Other-language
readers — a Python logger, tooling — read the same schema from the
published `unomsg` prefix's `share/`, not a per-repo copy.)*

Per-topic ring sizing:

- The default ring (≈64 KiB) is sized for small telemetry structs.
- Camera-rate or large-model topics need an explicit size hint.
  Oversize messages **truncate**, they do not error.
- Three knobs in order of preference: (1) per-deployment config,
  (2) per-workspace baseline / size-hint table, (3) ad-hoc at
  publisher construction. Prefer (1) for stable topics, (2) for
  family-wide defaults, (3) only for one-off cases.

<a id="descriptor-data-split"></a>
### 5.1 Send the descriptor on the bus; keep the bulk in shared memory it points at

For a large payload (a camera frame, a tensor, a multi-MB blob),
**don't put the bytes on the message bus** — put a tiny
*descriptor* on the bus and keep the bulk in shared memory both
ends map. The bus carries `{which buffer, sequence, dims,
timestamp}`; the pixels live in a SHM frame buffer (host) or a
shared device pool (GPU) and are never copied through the
bus/CPU per frame. A bus sized for telemetry will truncate or
choke on bulk; a per-frame copy of bulk is the throughput bug
you'll measure later.

This is the same shape at three layers:
- **Host SHM** — the frame-buffer pattern ([§8.8](ui-rules.md#texture-lifecycle)):
  one `memcpy` into a shared segment, consumers map a view, only
  the seq/dims travel.
- **GPU device memory** — share the *device* buffer cross-process
  so pixels never round-trip to host. On platforms where classic
  GPU IPC is unsupported, the VMM driver API works: export the
  device allocation to a POSIX **fd**
  (`cuMemExportToShareableHandle`), pass the fd **once** over a
  unix socket, the peer imports + maps the *same physical GPU
  memory*; from then on only descriptors flow over the bus. A
  pre-shared **pool** of N identical buffers (fd-shared once at
  startup) avoids per-frame handle setup.
- **Async, no per-frame sync** — the producer needn't
  `cuCtxSynchronize` (or host-block) every frame; signal a
  shared doorbell the consumer GPU-waits on
  (`cuStreamWaitValue32`-style), so the hand-off stays on the
  device timeline.

Two correctness traps when sharing device/SHM buffers:
- **The buffer is reused — clone before you keep it.** A pooled
  or JIT-output buffer is overwritten next cycle; a consumer that
  holds the descriptor past one cycle reads *current*, not the
  frame it meant (the temporal-pairing corruption from
  [§8.16](ui-rules.md#animation)'s neighbour). Clone if you retain.
- **Build the GPU context after `fork`,** not before — a context
  inherited across `fork` is invalid in the child.

The generalisable rule: **bulk payloads travel by reference
through shared memory, not by value over the bus; the bus moves
the descriptor.** Cross-compile the device-sharing layer with a
dynamically-loaded driver (dlopen `libcuda`) so a CUDA-less host
can build it ([§11](delivery-rules.md#cross-compile)), and validate on the target
([§9.5](#accelerators)) — device-IPC support is hardware-specific.

<a id="drain-per-wake"></a>
### 5.2 Drain the queue per wake — one read per tick silently drops a faster producer

A poll-loop consumer that reads **one item per iteration** keeps up
only if it wakes at least as fast as the producer emits. Against a
faster source it silently drops the rest: a reader that calls
`recv()` once per tick at 50 Hz against a ~900 frame/s bus forwards
~50 and **drops ~95 %** — and the low-rate items you actually need
statistically never arrive, so downstream marks itself invalid
(`canValid=False`) while everyone blames the *producer* ("the
device is silent").

Drain what's available each wake instead of one item: a short
**blocking** read for the first item (so the loop still sleeps when
the source is idle), then a **non-blocking drain loop** to empty
the backlog, batched into the one downstream message. Cadence
unchanged; throughput now matches the source.

The generalisable rule: **a queue/socket consumer drains all
available items per wake — reading one-per-tick caps intake at your
wake rate and silently drops a faster producer; the symptom reads
as "downstream invalid / source silent," but the loss is on the
reader side.** Confirm with a per-item arrived-vs-delivered count
(`bus=N cereal=M`), not a guess. See anti-pattern #64.

<a id="sole-publisher"></a>
### 5.3 One publisher per topic — and a base-class default that publishes is not inert

Single-publisher transports (msgq-style shm rings, DDS exclusive
ownership, retained-topic brokers) enforce that **each topic has
exactly one publisher** — a second registrant gets an error or evicts
the first. The contract is good; the trap is how a second publisher
*arises*: not by someone deliberately publishing the topic twice, but
by a **default implementation that publishes**. A framework base class
("no device? emit empty messages so consumers see liveness") is itself
a publisher — so a subclass that forgets **one wiring line** silently
falls back to the base, and the process now claims a topic that a
dedicated publisher owns.

We shipped this: a vehicle-interface subclass missed the one-line
class-attribute wiring its siblings had, so the **base** radar
interface ran — and the base emits the radar topic. The dedicated
radar publisher owned that topic, so the *core control daemon* died
with a publisher-collision error **on its first publish** — and only
once the radar publisher's run-gate came on (a bus going live), so the
bug sat **latent** through every test where the gate was off, then
"appeared" weeks later looking like a brand-new crash in the victim.

- **A base default that *does* something is a loaded gun.** A
  forgotten subclass wiring should fail loud (abstract method, missing
  registration error) or fall back to something **inert** — never to
  an implementation that publishes, sends, or actuates
  ([silent-default trap, §7](#silent-defaults), in class-hierarchy
  form).
- **Every topic names its one publisher** — that's the
  [new-topic checklist (§22)](delivery-rules.md#new-service-checklist) entry doing its
  job; a collision means two components each *believe* they own it.
- **The crash lands on the victim, not the culprit** — the
  late-binding process dies; the wrongly-publishing one looks healthy.
  And a **gated** publisher makes the collision latent: it bites only
  when both run-gates are finally on at once.
- **Opening a publisher *claims* the topic — even if you never send on
  it.** On a **claim-on-create** transport (a fresh writer-id stamped
  into the segment header at publisher construction), merely
  *constructing* a publisher revokes the previous owner — the real
  producer's next send then fails (`send failed (-1)` / `Closed`). So a
  **"keepalive publisher"** — one a process opens on its *input* topics
  just so a subscriber has something to attach to — is itself a
  collision: whichever process opens it *after* the true producer
  silently assassinates it (start-order / timing dependent, so it reads
  as a flaky transport fault). Modern subscribers **attach-or-create**
  the segment, so the keepalive is both unnecessary and harmful:
  **never create a publisher on a topic you don't send on.**
- **The diagnostic:** with the suspect process **dead**, sample the
  topics it should own — any topic still alive is being published by
  the collider; its identity tells you who. Faster than reading every
  candidate's code. (A daemon that dies `send failed (-1)` *shortly
  after another process starts* points the same way — a publisher
  collision on one of its **output** topics.)

The generalisable rule: **a topic has exactly one publisher, and a
base-class / default implementation that publishes is a publisher —
wire subclasses explicitly and make forgotten wiring fail loud or fall
back inert; on a claim-on-create transport, opening a publisher claims
the topic even if you never send, so never open one on a topic you only
consume (subscribers attach-or-create); diagnose a publisher collision
by sampling the dead process's topics (the one still publishing names
the collider), and expect gated publishers to make the collision
latent.** See anti-patterns #81, #92.

<a id="derive-the-registry"></a>
### 5.4 A bridge / forwarder derives its set from the live components, not a hand-maintained parallel list

A bridge, forwarder, or router that serves a *set* of things — the
topics a transport bridges, the subscribers a session rebinds, the
handlers a dispatcher routes to — must derive that set **from the actual
components** (enumerate them by type / interface), never from a **second,
hand-maintained list** that has to be kept in sync. A parallel list is a
standing invitation to a **silent-omission bug**: someone adds a
component and forgets the list, so the new one is never served — and
because nothing errors (it's just *absent*), the gap is invisible at the
forwarder and gets **misattributed to the source**.

We shipped this **twice, identically**: a GUI bridged only the topics its
session had rebound, and the rebind iterated a **hand-maintained tuple**
of subscriber names. A new `liveTracks` subscriber and, later, a
`parkingState` subscriber were each added but not put in the tuple — so
the GUI never opened the topic, the radar overlay and the parking pill
sat blank/"daemon SILENT" while the rig published at 10–20 Hz, and both
were debugged as **rig faults** (a dead daemon, a broken radar) when the
rig was fine.

- **Derive by type, not by name-list.** Build the served set by
  enumerating the real objects (`isinstance` the subscriber base, scan
  the plugin registry, reflect the annotated handlers) — so a newly-added
  component is **automatically** included and *cannot* be forgotten. The
  hand-list is a duplicate source of truth ([§28 one shared
  module](#reuse-critical-path) in registry form).
- **Make the omission impossible, don't checklist it.** "Remember to add
  it to the list too" is the same recurring miss as the [forgotten
  subclass wiring (§5.3)](#sole-publisher) and the
  [new-service checklist (§22)](delivery-rules.md#new-service-checklist) — a structural
  fix (auto-discovery) beats a discipline you have to re-apply every time.
- **The symptom points at the wrong layer.** A consumer showing *no data*
  (not an error) for a source that's healthily publishing is
  anti-pattern #1: check whether the bridge/router actually forwards it
  **before** blaming the producer.

The generalisable rule: **a bridge / forwarder / registry derives the
set it serves from the live components (by type or interface), never from
a hand-maintained parallel list — a second list silently drifts when a
component is added, drops it without erroring, and the missing data is
misattributed to the source; make inclusion automatic by construction.**
See anti-pattern #96.

---

<a id="process-supervision"></a>
## 6. Process supervision

> **When this fires** — adding or changing a long-running daemon — its gate, restart policy, reconnect, login dependency, or a manager that imports plugin modules.
>
> **Do** — gate predicate (never 'always run'); `Restart=always` for boot races; bounded reconnect that frees the old handle; isolate eager imports.

Every long-running process has a **gate predicate** registered with
the supervisor — never a bare default of "always run". Common gate
shapes: the process should only run when a feature flag is set,
when the platform reports a specific role, when a required device
file exists, when a peer service is healthy.

Why: a missing-hardware default of "always run" makes the
supervisor restart-loop the daemon forever and obscures the real
failure. A gate makes the absence visible.

When you add a process, also add a short inline rationale comment
on the gate if the choice isn't obvious. Future readers should not
have to reverse-engineer the predicate.

### Transient-resource daemons: retry with a loud warning

A daemon that opens a transient resource at startup (a TTY, a
network socket, a kernel device, a peer service) and treats the
opening as fatal will be **stranded for the entire session** if
the supervisor's `restart_if_crash` is False. We've shipped this
shape: missing group membership → `PermissionError` on
`/dev/ttyACM*` → daemon crashes once → no restart → silent dead
for the whole boot.

The right shape inside the daemon is:

```python
while not shutdown.is_set():
    try:
        fd = open(tty_path, "rb")
        break
    except (PermissionError, FileNotFoundError, OSError) as e:
        log.warning("opening %s: %s; retrying in 5 s", tty_path, e)
        shutdown.wait(5.0)
```

Loud WARNING per retry, never silent. The supervisor stays
healthy; the operator sees "device not ready" in the journal
instead of the daemon vanishing.

**`SerialException` gotcha:** `pyserial` wraps `PermissionError`
in `SerialException`. A bare `except PermissionError` doesn't
catch it. Match on the error message
(`"Permission denied" in str(exc)` / `"[Errno 13]" in str(exc)`)
or catch the wrapper class.

<a id="service-inventory"></a>
### 6.1 One declarative process table is the source of truth

Services are easy to lose track of: one gets added in a launch
script, another in a systemd unit, a third spawned ad-hoc by a
parent process, a fourth started by hand "just for now" and never
removed. After a while nobody can answer **"what is the full set
of services, and is what's *running* the set that's *supposed* to
run?"** That gap is where orphans, silent-dead daemons, and
"works on rig A not B" live.

The discipline is **one declarative table + reconciliation
against reality.**

**1. A single declarative table lists every service.** Not "the
union of three launch scripts" — one file the supervisor reads,
where each entry carries its gate predicate (§6) and a one-line
rationale. Adding/removing a service is a diff to this table, so
the full set is greppable, reviewable, and versioned (it travels
with the code; §7.5). If a service isn't in the table, it
shouldn't be running.

```python
# the table IS the inventory — one row per service
PROCS = [
  Proc("camerad",   gate=always,            why="capture"),
  Proc("imu",       gate=dev_exists("/dev/ttyACM1"), why="missing IMU ≠ restart-loop"),
  Proc("logger",    gate=param("RecordingEnabled"),  why="off in demo mode"),
  # ...
]
```

**2. Reconcile running-vs-declared — make the diff a command.**
The recurring failure is drift between the table and what's
actually alive (an orphan from a crashed parent, a hand-started
process, a daemon that silently died). Ship a probe that prints
the three-way diff:

```
declared but not running   → crashed / gated-off (is the gate right?)
running but not declared    → orphan / ad-hoc → kill or add to table
declared and running        → ok
```

This is the [diagnostic-first rule (§15.1)](delivery-rules.md#diagnostic-first) applied to the service set:
"is everything that should be up, up, and nothing else?" becomes
*one command*, not a mental tally. (We already lean on this —
an orphan-sweep at startup, §8.11, is the enforcement half;
the probe is the read half.)

**3. The supervisor owns lifecycle; nothing starts services
behind its back.** A service started outside the supervised set
(a stray `python -m …`, a leftover `&` in a script) is invisible
to the table and to restart/shutdown logic — it becomes the
orphan that posts duplicate offers or holds a stale resource.
One owner, one table, one shutdown path (§8.3 LIFO lifecycle).

**4. Removing a service is as deliberate as adding one.** Delete
the row, and grep for its name across the tree — its topic
consumers, its config keys, its bootstrap steps, its
documentation. A service removed from the table but still
referenced by a consumer is the mirror image of the
new-service-checklist (§22): the four things you owed on the way
in (name, consumer, gate, rationale) are the four things you
clean up on the way out.

The generalisable rule: **the set of services is declared in one
versioned place, and "what's running vs. what's declared" is a
command you run — never a list you keep in your head.** When the
two diverge, the table wins: reconcile reality to it (kill the
orphan, fix the gate, restart the dead daemon), or change the
table on purpose. See anti-pattern #31.

<a id="restart-policy"></a>
### 6.2 Restart policy: self-recover from clean-exit races; escalate the faults software can't clear

A restart policy has two edges that strand a service if you get
them wrong — one at each end of "should I restart this?"

**1. "Restart on failure" does not fire on a clean (zero) exit — so
a daemon that exits 0 on a transient race stays dead.** A manager
that starts before a dependency is ready and exits 0 (or uses
exit-to-restart as its own pattern) under `Restart=on-failure` is
**not** restarted: the unit goes `inactive(dead)` and waits for a
human. After an unattended reboot that's a rig that never comes
back — the stack silently never started. Use **`Restart=always`**
for any daemon whose normal or transient exits can include code 0;
a deliberate `stop` still stops it (the supervisor won't fight an
operator-requested stop). This is the unit-level form of the §6
"retry, don't strand" rule: the in-process loop handles a not-ready
*resource*; the restart policy handles a not-ready *process*. If
the unit file is **generated** (a bootstrap/onboard script writes
it), fix the template, not the live `/etc` copy — the next onboard
overwrites a hand-edit.

**2. Some faults are not software-recoverable — detect and
escalate, don't restart-loop forever.** A wedged USB device, a hung
sensor-module firmware, a GPU that stopped retiring fences
([§9.5 r3](#accelerators)) will not clear on a process restart, a
full stack restart, or even a bus re-enumerate — only a
**power-cycle** (a reboot, or a physical replug for
separately-powered hardware) clears it. A supervisor that keeps
software-restarting against a hardware hang spins forever and buries
the real cause under restart noise. The right shape: distinguish
*recoverable* (retry — the §6 transient-resource loop) from
*needs-power-cycle* (a device still 0 Hz after N restarts), and on
the latter **say so loudly** — in the journal and on any operator
surface ("sensor firmware hung — reboot required") — instead of
retrying invisibly. `Restart=always` (rule 1) is what then makes
the reboot *self-heal*, once an operator or a hardware watchdog
triggers it. But a reboot here is the **deliberate last resort
after diagnosis confirms the fault** — not a reflexive
make-it-go-away move that wipes the evidence; capture the volatile
state and try the narrowest recovery first ([§15.7](delivery-rules.md#no-reflexive-reboot)).

**3. `Restart=always` is still defeated by the *start-limit* — a
fast-flapping unit wedges in `failed` indefinitely.** systemd's
default `StartLimitBurst=5` / `StartLimitIntervalSec=10s` is a
give-up guard: restart 5 times in 10 s and systemd **stops
restarting** and leaves the unit `failed`, so the "always-on"
service is now permanently **DOWN** — `Restart=always` looks like it
covers this but doesn't. For a service that *must* keep trying (a
stack whose dependency flaps, a manager that crash-loops while a
resource comes up), set **`StartLimitIntervalSec=0`** to disable the
limiter — and put it in the **`[Unit]`** section, not `[Service]`,
where it is silently no-op'd. Pair it with a **generous
`TimeoutStopSec`**: a slow stop (a child hung in a codec / DMA wait)
that overruns the timeout escalates to `SIGKILL → failed(timeout)`,
and each such failure burns the restart budget that trips the limit
in the first place. And the **first diagnostic** for "stale / frozen
/ no updates" from a supervised service is **`systemctl is-active
<unit>`**: a `failed` / `inactive` unit *is* the whole story — but
stale values linger in `/dev/shm` and a rate probe reads the **last
live values off the dead producer**, misleading you into the wrong
subsystem (derive liveness from the live link, [§15.10](delivery-rules.md#liveness-from-data);
[diagnostic-first](delivery-rules.md#diagnostic-first)).

The generalisable rule: **choose the restart policy for how the
process actually exits — `always` when a clean exit can be
transient, so a reboot self-recovers — and bound software retries
by recoverability: a fault only a power-cycle clears must be
surfaced for escalation, never restart-looped in silence.
`Restart=always` is necessary but not sufficient: the default
start-limit wedges a fast-flapping unit in `failed` forever, so a
must-always-run service disables the limiter
(`StartLimitIntervalSec=0` in `[Unit]`) and keeps `TimeoutStopSec`
generous, and you diagnose a "stale data" report with `systemctl
is-active` first.** See anti-patterns #45, #120.

<a id="always-on-contract"></a>
### 6.3 The always-on service contract: clients don't stop it, and reconnect across its restarts

A service that is supervisor-owned and **`Restart=always`**
([§6.2](#restart-policy)) is declaring "I am meant to be up at all
times." Everything *around* it has to honour that, or the always-on
guarantee is quietly broken from the outside:

- **A client / tool / GUI must never `stop` it as a side effect of
  its own teardown.** A "cancel," an app-exit hook, or a superseding
  relaunch that runs `systemctl stop <always-on>.service` takes the
  service down *with the client* — and an **explicit stop also
  defeats `Restart=always`** (the supervisor won't fight an operator
  stop), so it stays down until someone notices. We shipped exactly
  this: closing the GUI stopped the rig's driving stack. On your own
  teardown, **detach** (drop the log tail / the connection); operate
  on the service with **restart/reload**, never `stop`. Reserve
  `stop` for a deliberate, explicit operator action — never a
  cleanup path.
- **A long-lived client must self-heal across the service's
  restarts.** The peer *will* restart — a redeploy, an onboard, a
  rig reboot, a crash. A client that assumes one successful connect
  goes permanently stale the first time the service bounces (one
  offer made, session closed, nothing reconnects until an operator
  re-triggers). Make reconnect automatic: a health tick re-spawns a
  dead worker / re-establishes the session while the resource is
  still wanted, **rate-limited with a backoff** so a genuinely-down
  peer doesn't thrash ([§10.2 capped recovery](#capped-recovery)).
  Don't fight a lower layer that already reconnects — cover only the
  gap it doesn't (e.g. the whole worker process exited).

The generalisable rule: **an always-on service's lifecycle belongs
to its supervisor — peers operate on it with restart/reload and
never `stop` it from a teardown path (an explicit stop defeats
`Restart=always`), and clients reconnect automatically across its
restarts instead of going stale after one connect.** See
anti-pattern #51.

<a id="bounded-reconnect"></a>
### 6.4 Reconnect is bounded and releases the old handle — a leaked capped resource freezes everyone

A long-running subscriber/client that re-opens a connection on
silence (a dead topic, a dropped socket) has two ways to turn a
transient blip into a system-wide outage:

**1. Reassigning the local handle does NOT release the shared
resource.** IPC reader slots, file descriptors, DB-pool
connections, semaphores are **capped, shared** resources; the OS /
broker frees the slot only when you *close* it — not when you
reassign the variable that referenced it. `self.sock = sub(...)` in
a loop leaks a reader every iteration ("reassigning frees the fd"
frees the *local* fd, not the broker's slot). Close the old handle
before opening a new one.

**2. Unbounded reconnect-on-silence is the leak engine — cap it,
then escalate to restart.** Re-subscribing every few seconds for
any silent topic, forever, races a capped resource to exhaustion.
And exhaustion here isn't a local error you'd notice — many brokers
**evict or block *all* users** of a resource when the cap is hit
(one shared bus evicts every reader of a topic at its 16th), so one
leaky client **freezes the whole stack**, healthy consumers
included. Cap the retries per silence *episode* (reset on data);
past the cap the resource is genuinely down, so let the supervisor
restart the process ([§6.2 `Restart=always`](#restart-policy)) —
don't open unbounded sockets. (Capped recovery,
[§10.2](#capped-recovery), applied to reconnection.)

**The symptom lies about its location.** Capped-resource exhaustion
presents as "everything froze" — every *subscriber* dead while
publishers and an unrelated control path are fine — and "a restart
fixes it" because the restart recreates the resource (cap resets).
Attribute it to the leaker, not the frozen victims
([§9.9](#shared-resource-contention), [§15.7](delivery-rules.md#no-reflexive-reboot)),
and don't trust a proxy signal (an mmap'd shm file's mtime doesn't
bump on write — a "frozen mtime" is not a freeze).

The generalisable rule: **a reconnect/resubscribe loop releases the
old handle (reassigning a variable doesn't free a shared slot) and
caps its attempts per episode then defers to restart — an unbounded
loop leaks a capped shared resource to exhaustion, which many
brokers turn into an everyone-freezes outage, not a local error.**
See anti-pattern #62.

<a id="service-not-login-bound"></a>
### 6.5 A long-running service must not depend on an interactive login — its user IPC is reaped on logout

A service that runs **as a user** and relies on that user's POSIX
IPC — `/dev/shm` (shared buffers, msgq), POSIX message queues,
semaphores — breaks when that user's last interactive session ends.
systemd-logind's default **`RemoveIPC=yes`** unlinks all of the
user's IPC objects on final logout. The failure is nasty because it
*looks* fine on one side: existing publishers keep writing (Linux
`unlink()` doesn't break a live `mmap`), but every **new**
subscriber `open()`s a **fresh, empty inode** at the same path — so
the wire is alive at the producer and silent to anything that
reconnects, until the service restarts.

- **Decouple the service from the login.** Run it as a real
  **system** service, or `loginctl enable-linger <user>` so logind
  never reaps that user's IPC. A daemon's liveness must not hinge on
  someone being SSH'd in.
- **Diagnose by inode, not by "is it running."** A publisher whose
  shm shows `(deleted)` in `/proc/<pid>/map_files` is writing to an
  orphaned inode; a subscriber-side `stat` shows a newer one.
- **Self-heal if you can't guarantee linger:** the publisher
  `stat`s its shm path and, on an inode change, drops and re-binds
  the socket on the live inode.

The generalisable rule: **a long-running service must not depend on
an interactive login — under logind's default `RemoveIPC=yes` the
user's `/dev/shm`/IPC is garbage-collected at last logout, silently
splitting old publishers (deleted inode) from new subscribers (fresh
inode); run it as a system service or `enable-linger`, and diagnose
by inode.** See anti-pattern #63.

<a id="eager-import-isolation"></a>
### 6.6 A supervisor that imports every registered module at startup has no load-time fault isolation

Many process managers / plugin hosts build their table by
**importing every registered module at init** — to read its config
and resolve its entry point — *before* any run-gate (`should_run`) is
consulted. That makes the import step a **single point of failure for
the whole table**: one module that fails to import — a stale path
after a rename, a missing optional dep, a syntax error — raises at
init and **takes down the entire manager**, including healthy,
unrelated services. The run-gate does **not** save you: a process
gated *off* (it would never run) is still *imported*, so its broken
import is exactly as fatal as a running one's.

We shipped this: after a package rename, one registered process
module kept a stale `import <oldpkg>…` (a sibling module was already
migrated). It was gated off by default — so "it never runs" *felt*
safe — but the manager imports every registered module at startup, so
the `ModuleNotFoundError` was fatal at init. The whole stack
crash-looped for hours (restart every few seconds, every subsystem
down) over one import line in a process that wasn't even supposed to
run.

- **Diagnose at the load layer, not the restart spam.** A stack-wide
  crash-loop (`NRestarts` climbing, `status=1`, restart every few
  seconds, *everything* down) usually means the manager died at
  **init**, not that N services each crashed. Read the manager's
  **first traceback**, not the supervisor's restart messages
  ([looks-like the busiest subsystem, is one line](delivery-rules.md#diagnostics)).
- **A registered module is imported even when gated off.** Don't
  reason "that feature is disabled, so its code can't break us." After
  a rename / refactor, sweep the old name across **every** registered
  module, not just the ones currently running
  ([§18.2](delivery-rules.md#migration-completeness)).
- **Isolate the import where you can.** If the host imports each
  module in a `try` and disables only the failing one (loudly), a bad
  module degrades to **one** dead service instead of a dead stack —
  the load-time analogue of [per-stream isolation §10](#per-stream-isolation).
  At minimum, evaluate the run-gate *before* importing an optional /
  dark module.

The generalisable rule: **a supervisor that eagerly imports every
registered module at startup turns any single import failure into a
whole-table outage — and a run-gate doesn't protect you, because a
gated-off module is still imported. After a rename, sweep the old
name across every registered module; diagnose a stack-wide crash-loop
at the manager's init traceback; isolate per-module imports so one bad
module can't kill the rest.** See anti-pattern #71.

<a id="resident-vs-scheduled"></a>
### 6.7 A job that must be "always on" is a resident supervised service, not a scheduled one-shot

A periodic task you want **always available — up on boot,
self-recovering, never "off"** — is the wrong fit for a calendar/cron
one-shot (a `cron` line, a systemd `*.timer`, a launchd
`StartCalendarInterval`). A scheduled one-shot has **no resident
process between fires**, and that costs you three things:

- **It doesn't come up on boot/login.** It waits for the next
  scheduled instant; a fire due while the machine was asleep or off is
  simply skipped, not made up. Nothing runs until the clock next matches.
- **It can't self-heal.** There is no live process to keep alive —
  "restart it if it dies" has nothing to act on. Each fire is a fresh,
  unsupervised one-shot.
- **It reads as dead.** Between fires there is no PID; `status` / a
  health check shows it **idle / not running**, which looks *broken* to
  anyone checking — even when it is behaving exactly as designed.

If the requirement is "always running, starts itself, recovers if it
dies," make it a **resident supervised service**: a process that loops
(`do-work → sleep → repeat`) under a supervisor that **starts it on
load/boot** (`RunAtLoad`, or `systemctl enable` + `WantedBy`) and
**restarts it if it exits** (`KeepAlive`, or `Restart=always` —
[§6.2](#restart-policy)). Keep a failed work-cycle from killing the loop
(log it, continue to the next iteration); only an actual crash exits,
and the supervisor brings it straight back — so there is always a live
process, it comes up after a reboot, and it self-heals.

Choose by the requirement, not by which is fewer lines:

- **Resident service** — must be live, boot-started, self-healing, or
  "never idle": a control/telemetry daemon, a watcher, an uptime-keeper.
- **Scheduled one-shot** — a missed run is harmless and a
  no-live-process resting state is fine (a nightly report, a periodic
  cleanup). Here **"idle between fires" is the healthy state** — don't
  misread it as broken, and don't bolt on a keep-alive that would just
  fire it in a tight loop.

The generalisable rule: **match the lifecycle model to the requirement.
An "always-on / starts-at-boot / self-recovering" job is a resident
process that loops under a supervisor which starts it on load and
restarts it on exit ([§6.2](#restart-policy)) — not a calendar/cron
one-shot, which has no live process between fires, so it can't
boot-start or self-heal and its idle resting state reads as dead.** See
anti-pattern #86.

<a id="singleton-process"></a>
### 6.8 A process that must be a singleton enforces it — a silent duplicate makes the system stale

Some processes must run **exactly once** per host / resource — the
publisher of a topic, the writer of a shared throttle / config slot, an
adaptive controller, a device-owning daemon. Start a **second** instance
by accident — a relaunch that didn't stop the old one, an operator who
ran it again, a supervisor that double-spawned, a session that exited
without cleanup — and the two don't crash; they **both run and both write
the shared state**, which now flips between their outputs. Nothing errors;
the system just goes **stale / flapping**, and you debug the *victim*
(the value that won't hold) instead of the duplicate.

We've shipped this shape repeatedly: a dead GUI session left an
adaptive-throttle value the live rig kept obeying; two viewer sessions
each drove the bitrate budget; a keepalive publisher claimed a topic the
real producer owned ([§5.3](#sole-publisher)). The common cause is **no
single-instance guard** and **no reaping of the predecessor**.

- **Enforce single-instance at startup.** Take an exclusive lock (a
  `flock`'d pidfile, an `O_EXCL` lockfile, a bound singleton socket, or —
  on a claim-on-create transport — the topic claim itself,
  [§5.3](#sole-publisher)). A second start either **becomes a no-op**
  (one is already healthy) or **takes over and reaps the old one** —
  never silently coexists.
- **The singleton is the *role*, not the binary.** A *different
  implementation* of the same role is still a duplicate — a rewrite
  deployed beside the legacy it replaces, a second daemon that owns the
  same device / encoder / bus / IPC ring / actuation authority. A lock
  keyed on the binary's own name won't catch a foreign implementation, so
  guard on the shared **resource** they contend for (the device handle,
  the topic claim [§5.3](#sole-publisher), an actuation-authority lock).
  Coexistence is safe **only when the implementations own disjoint
  resources**; the moment they share one, "each under its own manager" is
  a collision, not isolation — the symptom lands on the *victim* (no
  device, a consumer stuck at 0 Hz, load pegged) while every producer
  looks healthy, and if both can actuate it is a safety hazard. The
  coexistence window of a clean rewrite ([§1](#rewrite-clean-break)) is
  exactly when this bites.
- **Claim the role by *disabling* the others, not just stopping them —
  every scope, the whole set, at two enforcement points.** "Stopped"
  isn't "gone": a competitor left merely stopped but still **enabled**
  re-starts on the next boot and re-grabs the resource. So at **onboard**
  stop **and disable** each other implementation in **every** service
  scope (system *and* per-user — a host may use either), and reap orphaned
  holders **by exe-path / pattern, not just the unit** (a supervisor's
  children outlive the unit stop and keep holding the device). Disable the
  **whole set** of competitors **derived from the stack registry**, never
  a hand-kept parallel "others to disable" list — that list silently drops
  the next stack you add ([§5.4](#derive-the-registry)) and the forgotten
  one keeps fighting. Apply the *same* claim at **two points**: a one-time
  privileged cutover so the choice **survives reboot** (persistent
  disable), and a **runtime claim on every activation** that frees the
  resource *now* and re-asserts if a competitor crept back — degrading
  gracefully (without privilege the reap still frees the device, even if
  the persistent disable needs the one-time privileged step).
- **Reap the predecessor / orphans on startup.** A new instance sweeps
  stale state from a dead prior — orphaned workers, a pidfile whose
  process is gone, a `/tmp` slot/segment a crashed writer left
  ([§8.11 orphan sweep](ui-rules.md#orphan-sweep),
  [§6.5 IPC](#service-not-login-bound)). Pair it with **freshness on the
  read side** ([§7.6](#stale-pushed-override)): a consumer expires a
  pushed value whose writer went away, so a leaked duplicate's output
  can't drive forever.
- **Retry the claim on startup — a just-reaped predecessor's release
  lags the restart.** A supervisor that reaps the old instance and
  starts a new one a second later **races the OS**: the loopback **port**
  is still in `TIME_WAIT`/not-yet-freed, the `flock`/`O_EXCL` lock or
  pidfile not yet released, the device handle not yet closed. A
  **one-shot** claim then fails (`EADDRINUSE` / lock-held) and the new
  instance exits with *"another instance is already running"* when in
  fact **none is** — so the service stays **DOWN** after a restart.
  Retry the claim over a **bounded window** (a few seconds) so a
  transient post-reap release is waited out; a **genuinely-live** peer
  never releases, so the claim still fails after the window and the
  singleton invariant holds. Restart is faster than teardown —
  [`Restart=always`](#restart-policy) makes this the common path, not a
  rare one.
- **Diagnose a stale/flapping shared value by counting the writers.** A
  setting that won't hold, or output that alternates between two
  near-values, is the duplicate-writer signature — `pgrep`/`ss` the
  instances and the topic's publishers before tuning the value or
  blaming the consumer.

The generalisable rule: **a process that must run once per host/resource
guards single-instance at startup (a lock / claim, take-over-and-reap not
silent-coexist) and reaps the predecessor's orphaned state — a second
silent instance writes the same shared state as the first and the system
goes stale or flaps, with the symptom landing on the victim, not the
duplicate. The singleton is the **role**, not the binary — a different
*implementation* of the same role (a rewrite beside the legacy) is also a
duplicate, so onboard one by **disabling (not just stopping) and reaping
every other implementation** — across scopes, for the whole
registry-derived set, at both the privileged cutover and the
per-activation claim — guarding on the shared resource, never one binary's
name. Retry the claim briefly on startup, too — a just-reaped
predecessor's release (a `TIME_WAIT` port, a lock, a pidfile) lags the
restart, so a one-shot claim misreads the lag as a live peer and leaves
the service falsely "already running" / down.** See anti-patterns #102,
#108, #112.

<a id="cooperative-stop"></a>
### 6.9 Stop a worker cooperatively before you kill it

Tearing down a worker — a thread, a subprocess, an async task — by
**immediately** signalling `SIGTERM`/`SIGKILL` (or dropping its handle so
the runtime aborts it) **mid-operation** is how a clean shutdown turns
into a **crash**. A worker caught mid-step is usually holding something
that doesn't survive a forced kill: a native/codec context (a GPU decoder
mid-frame), a `dlopen`'d library's state, a lock, a half-written buffer, a
DMA in flight. The kill races the operation and **segfaults or corrupts
shared state** — and because the symptom is a crash *on exit*, it reads as
"flaky teardown" rather than "we killed it at a bad moment."

Teardown is **two phases**, not one:

- **Phase 1 — cooperative stop.** Signal *intent* (set a `stop_event` /
  cancellation flag the worker's loop checks each iteration; close the
  input channel) and **wait, bounded**, for the worker to reach a safe
  point and exit on its own — releasing its codec / handle / lock as it
  goes. This is the path that should normally complete.
- **Phase 2 — forceful escalation.** Only if it doesn't exit within the
  bound do you escalate to `SIGTERM`, then `SIGKILL`. Forcing is the
  *fallback for a hung worker*, never the first move.

Skipping phase 1 makes **"stop" itself the bug**: the teardown path
crashes a worker that would have exited cleanly a few milliseconds later.
This is the shutdown twin of the [§6.4](#bounded-reconnect) handle
discipline and pairs with orphan-reaping
([§6.8](#singleton-process)) — reaping cleans up after an *ungraceful*
exit; this *avoids* the ungraceful exit in the first place. (A worker that
genuinely can't reach a safe point in the bound is a design smell — give
it a checkpoint, don't just kill harder.)

The generalisable rule: **stop a worker (thread / subprocess / task)
cooperatively before forcing it — signal a stop flag and wait a bounded
time for it to exit on its own (releasing its codec / handle / lock),
escalating to `SIGTERM`/`SIGKILL` only if it hangs; a forced kill
mid-operation crashes or corrupts because the worker is holding a native
resource, so an immediate kill makes the teardown path itself the fault.**
See anti-pattern #116.

<a id="device-access-in-unit"></a>
### 6.10 A daemon's access to its devices is provisioned in the unit, not your shell

A daemon that opens a device — a serial / USB TTY (`/dev/ttyACM*`), a CAN
socket, an i2c / GPIO node, a GPU — needs OS **permission** to do so, and
that permission comes from the **service unit**, not from your interactive
session. A supervisor starts the process with only the unit's `User=` /
`SupplementaryGroups=` / capabilities — so "I'm in the `dialout` group"
*interactively* does **not** mean the **service** is. A device node is
typically `root:<group>` mode 660 (serial TTYs are `root:dialout`); a
service whose user lacks that group gets `PermissionError [Errno 13]` at
open, and the **sensor it drives goes dark**.

The failure is doubly hidden, which is why it gets misdiagnosed as bad
hardware:

- **Silent at the device.** A daemon that catches the open error and
  sleeps, or crashes once and isn't restarted (a `restart_if_crash=False`
  / no-`Restart` supervisor strands it, [§6.2](#restart-policy)), produces
  **no data** — and the consumer reports "sensor invalid / no GPS / no
  frames," so you debug the *hardware* instead of the *permission*.
- **The error is often wrapped.** A library may re-wrap the OS error in
  its own type (pyserial wraps `PermissionError` in `SerialException`), so
  a narrow `except PermissionError` misses it — match the cause / errno.

- **Provision the access in the unit.** Grant the group
  (`SupplementaryGroups=dialout`), the capability, or a udev rule / device
  ACL the daemon needs — in the **version-controlled unit**, deployed with
  the service. Verify against the *service's* environment (`systemctl show
  -p SupplementaryGroups`, the daemon's own open), not your shell's.
- **A device-open failure surfaces loudly.** Retry-with-WARNING (a real
  recovery path, [§18.9](delivery-rules.md#inject-the-fault)) and/or a
  persistent operator "can't reach device" indicator
  ([§3.5](#validate-safety-input)) — never a silent sleep or a one-shot
  crash into a stranded daemon (pair with `Restart=always`,
  [§6.2](#restart-policy)). Sibling of [§6.5](#service-not-login-bound):
  the service's environment is the unit's, not the operator's shell's.

The generalisable rule: **a daemon's access to its devices
(serial / USB / CAN / GPIO / GPU) is provisioned in the service unit — the
right group / capability / udev ACL — because the service runs with the
unit's permissions, not your interactive shell's; and a device-open
permission failure surfaces loudly (retry + WARNING / operator indicator),
never a silent sleep or a one-shot crash that strands the daemon, or a
missing group reads as "sensor invalid / no data" and you debug the
hardware.** See anti-pattern #119.

---

<a id="silent-defaults"></a>
## 7. Silent-default traps

> **When this fires** — reading a config / Param / schema field (a `getattr`/default), reaching for an env var, or polling a pushed override.
>
> **Do** — distinguish *absent* from *false* at the boundary; **never an env var** to select behaviour; expire a stale pushed value to a safe default.

A whole family of bugs reduces to: code reads a field that doesn't
exist (or is unset) and gets a **falsy value back instead of an
error**. The fallback then silently picks the wrong branch. Three
shapes show up across the codebase; all three have the same fix.

<a id="params-trap"></a>
### 7.1 KV / Params stores

Key-value config stores in this house return a **falsy default**
(`False`, empty bytes, empty string) for unset keys — **not
`None`**. The naive

```python
raw = params.get_bool("UseFeature")
if raw is None:        # never fires on fresh install
    return True
return raw
```

silently disables a "default ON" feature on every fresh install.
The fix is **accept only explicit opt-out values**:

```python
def use_feature() -> bool:
    env = os.getenv("UNO_FEATURE")
    if env is not None and env.strip().lower() in ("0", "false"):
        return False
    raw = params.get("UseFeature")        # bytes / str / None
    if raw in (b"0", b"false", "0", "false"):
        return False
    return True
```

Apply this every time you add a "default ON" Param. Document the
explicit opt-out values inline.

<a id="schema-getattr-trap"></a>
### 7.2 `getattr(obj, name, default)` against an evolving schema

The same shape bites schema reads. Suppose a capnp / protobuf /
dataclass struct moved a field from the top level into a nested
sub-message between versions. Code that reads it as:

```python
flags = getattr(evta, "flags", 0)      # silently 0 forever
```

returns `0` for **every** call, because the top-level `flags`
attribute genuinely doesn't exist anymore. The reader didn't
fail-fast; it returned a plausible-looking zero. Downstream logic
that uses `flags & KEYFRAME` then treats every frame as
non-keyframe, with seconds-of-black-video kind of consequences.

Two ways to avoid this:

1. **Don't `getattr(..., default)` against a schema you control.**
   Read the field directly. `AttributeError` at the call site is
   the right symptom — it tells you the schema moved.
2. If you must use `getattr` (across versions, across vendors),
   make the default sentinel non-falsy:

   ```python
   _MISSING = object()
   raw = getattr(evta, "flags", _MISSING)
   if raw is _MISSING:
       raise SchemaMismatch("evta.flags moved — check evta.idx.flags?")
   ```

<a id="function-default-trap"></a>
### 7.3 Default arguments that swallow a missing source

```python
def render(rpy=(0, 0, 0)):
    ...
```

A caller that forgot to pass `rpy` doesn't get an error — they
get the wrong default. Origin-pose defaults for
sensor-extrinsics, identity transforms for calibration matrices,
zero-rpy for "use the rig" geometry — all examples of "the
default looks valid so the bug aims the warp at the wrong
patch." Either require the argument (no default), or have the
caller pass an explicit sentinel and fail loudly when it's
absent.

<a id="override-flag"></a>
### 7.4 Internal "was this overridden?" uses a companion bool

The same shape recurs inside state machines, not just at config
boundaries. A picker / negotiator / fast-path selector writes a
field on a parent state object and a consumer reads it later.
If the empty / zero / falsy value is also a **valid override**
("the picker says: drop the proxy-jump, go LAN-direct"), an
`or`-chain reduction silently picks the fallback:

```python
# WRONG — "" conflates "no override" with "override to empty"
self._fast_proxy_jump: str = ""
...
proxy_jump = self._fast_proxy_jump or t.proxy_jump
# picker says "" → fallback fires, vehicle still routes via VPS

# RIGHT — companion bool distinguishes the two
self._fast_proxy_jump: str = ""
self._fast_proxy_jump_set: bool = False
...
proxy_jump = (self._fast_proxy_jump
              if self._fast_proxy_jump_set
              else t.proxy_jump)
```

Reset both fields together on every fresh selection so a stale
flag doesn't leak across runs.

<a id="env-var-config"></a>
### 7.5 Never select behaviour, a code path, or a device with an env var

**Standing rule — default to NO new environment variable.** This is
the rule most often rationalized around ("it's just a quick toggle"),
so hold it as a hard default: **do not introduce an environment
variable to change anything about how the program behaves** — a
feature, a config value, a forwarded-topic set, a compute device, a
pipeline variant, a resolution, a tuning constant, a tier. If you
are reaching for an env var, you have the **wrong design** — stop
and use option 1 or 2 below. A behavioural env var is a deliberate,
operator-approved exception with a written reason, **never** a
convenient default and never "just for now." When unsure,
**ask** ([§24](delivery-rules.md#when-in-doubt)); do not quietly add one.

If flipping a value changes *which code runs* — a feature, a
forwarded-topic set, a compute device, a pipeline variant — it does
**not** belong in an environment variable. Env vars rot: set in a
shell or a systemd unit on one rig and forgotten, they differ
silently between machines, don't travel with the code or the
artifact, and turn every later bug into a "works on rig A, not B"
hunt. This applies to **config** (which peers, which limits) *and*
to **code-path / device switches** (`DEV=CUDA`, `CAMERAD_RAW_VIPC`,
a CPU-vs-GPU toggle) alike.

**Do this instead:**

1. **Encode the decision in code, as a function of hardware /
   role.** The strongest form — the choice is derived, not set:

   ```python
   # device.py — one resolver, derived from the real target (§9.8)
   def preferred_device():
       if is_jetson(): return "CUDA"
       ...
   USE_TINYGRAD_WARP = JETSON and not TICI   # a fact about the board, not a knob
   ```

   "Which code runs" becomes a property of *what this machine is*,
   computed once, identical on every rig of that class — nothing
   to set, nothing to forget.

2. **Or a checked-in / on-disk config file** read through a typed
   loader — a service TOML, a Params-style store, a committed
   `<rig>.config.json`, a calibration overlay. Greppable,
   diffable, reviewable, versioned with the code.

The env-var version (e.g. a `BRIDGE_OUT` listing the topics a
bridge forwards) is a bad idea because:

- **It drifts silently across rigs.** Each rig's env is set
  independently — a shell profile, a systemd unit, a launch
  script, a stray `export`. Rig A forwards `carControl`, rig B
  doesn't, and nothing in the repo records the difference. This
  is the engage-pill bug ([anti-pattern #1](anti-patterns.md))
  and the version-drift bug ([#29](anti-patterns.md)) in config
  form.
- **It's invisible and un-auditable.** The value lives in process
  environment, not in a file you can read, diff, or commit. "What
  is this rig actually configured to do?" has no answer you can
  grep. You can't review it in a PR, can't diff it across rigs,
  can't see it in the git history.
- **It's not versioned.** A config file is committed and travels
  with the code that reads it; an env var is set out-of-band and
  has no version, no history, no migration path.
- **It silently falls back.** An unset behavioural env var doesn't
  error — it takes some default, so a rig that simply never had
  the var set behaves differently with no signal. (Same family as
  the silent-default traps above.)

**Instead:** put per-deployment behavioural config in a
**committed, versioned config file** read through a config API —
a service TOML, a Params-style store, a checked-in
`<rig>.config.json`. The value is then greppable, diffable across
rigs, reviewable in a PR, and travels with the code. If a value
must vary per rig, the *file* varies per rig (and is committed
per rig / selected by rig-id), so the variation is explicit and
auditable — not a property of whoever last edited a shell
profile.

**The only env vars that are OK** — and even these, prefer not:

- A **throwaway debug nudge** for a *single local run* — never
  committed, never the default any code path takes, **removed
  before merge**. The moment it survives into a commit, a launch
  script, or a systemd unit, it is config — and it is banned. (So:
  not a way to ship a behaviour, just a developer poking a running
  process.)
- A pure **location / secret the deploy system injects** — a path,
  a token: a *where*, never a *what*. Prefer the platform's
  resolver over a hardcoded path (anti-pattern #25).

Anything that selects *behaviour* — even one boolean, even
"temporarily" — is **not** on this list. If you think your case is
the exception, it almost certainly isn't: take it to the operator,
don't decide it yourself.

**The test (memorise this):** *if flipping the value changes which
code runs, it belongs in code keyed on hardware/role — not in the
environment.* Or, from the operator's seat: *"if two rigs behave
differently, can I see why by reading committed files?"* If the
answer needs inspecting a live process's environment, it's in the
wrong place. Migrate known env switches to code-derived; don't add
new ones. See anti-pattern #30.

<a id="stale-pushed-override"></a>
### 7.6 A pushed control value needs a freshness contract — or a dead writer is obeyed forever

The silent-default's **time-domain cousin**. A transient writer — a
GUI session, a client, an adaptive controller — pushes a control
value through a side channel (a `/tmp` file, a Param, a shared-memory
slot) and a consumer **polls** it. If the consumer reads the **value
but never its freshness**, the setting outlives its writer: when the
session disconnects, crashes, or is reaped *without cleanup*, the
consumer **keeps enforcing the dead value forever** — there's no
active sender, yet the override still stands.

We shipped this: an adaptive-bitrate controller over-throttled a
video encoder during a network blip, then its session was reaped. The
encoder's watcher polled the throttle file **by value only**, so a
"8 fps / 80 kbps" written ~5 minutes earlier by a long-dead session
pinned the live encode at 6 Hz — while the rig was completely healthy
and *no operator was connected*. It read as a rig fault; it was a
ghost setting.

- **Stamp freshness and expire it.** Treat a pushed value as
  perishable — check its `mtime` / heartbeat / sequence and, once it
  goes stale (untouched > N s), fall back to a **safe default** and
  let the source re-adapt from a clean baseline. Don't obey a value
  that carries no recency.
- **Don't rely on the writer cleaning up.** A clean-disconnect reset
  is nice, but a writer that *crashes* or is *reaped* never runs its
  cleanup — so the **consumer-side TTL is the robust half**, not the
  writer-side reset.
- **Diagnose it as staleness, not a fault.** A consumer stuck at an
  overridden setting while its producer is healthy and *no writer is
  active* is a stale override — find the side-channel value and its
  **age** before chasing the consumer.

The generalisable rule: **a control / override value pushed over a
side channel must carry a freshness contract (mtime / heartbeat /
TTL); a consumer that reads the value without its recency obeys a dead
writer forever — expire a stale override to a safe default and let the
source re-adapt.** This is the latest-wins / latency-bounded principle
of a control feed ([§3.4](#control-feed-freshness)) applied to the
override channel. See anti-pattern #73.

### Common thread

Falsy defaults are dangerous because falsy values are usually
valid. The fix is the same in all four cases above: **distinguish
"absent" from "false"** at the boundary where the value is
read, not deep in the consumer that has no way to tell them
apart.

---

<a id="ui-architecture"></a>
## 8. UI architecture

> **When this fires** — any interactive-surface change.
>
> **Do** — read **[`ui-rules.md`](ui-rules.md)** — §8 and all its sub-rules live there (snapshot pattern, wire-state mirror, masking, the house design language).

The UI rules — **25 sub-rules** across six themes (layer boundary;
snapshot / data flow; lifecycle & resources; look & locale; operator
truth & safety; layout & device fit) — live in their own file:
**[`ui-rules.md`](ui-rules.md)**. They are the largest single section
and read better standalone. The numbering is unchanged: §8 and every
§8.N / slug anchor live there, so existing references still resolve.

**Read [`ui-rules.md`](ui-rules.md) before any UI change.**

---

<a id="rust-vs-python"></a>
## 9. Choosing Rust vs. Python

> **When this fires** — picking a language for new code, a new service, or a refactor.
>
> **Do** — **default to Rust**; Python only for UI / ML-tensor glue / a true one-off script; reusable code goes in **`unolib`**.

**Org default: Rust.** New code is **Rust unless you can name why it
can't be** — services, daemons, libraries, codecs, transforms,
controllers, CLIs, and anything reusable (which goes in
**`unolib`**, [§28](#reuse-critical-path)) default to Rust, for the
speed *and* the compile-time safety (no data races, exhaustive error
handling, no silent `None`). A *new service in Python* is the thing to
stop and justify — or write in Rust.

**Choosing Python is the exception, and you justify it.** It's the
right tool only where the toolchain forces it or the work is
genuinely throwaway:

- **UI** bound to a Python framework (Qt / imgui).
- **ML / tensor runtime glue** — the model stack is Python.
- **A true one-off script** — not a "script" that grows into a
  service.

Anything outside those defaults to Rust. "Feels fine in Python,"
"faster to type," "we already have a Python file here" are **not**
justifications.

Two points hold regardless of language:

- **Don't churn-rewrite *working* Python on a hunch.** The default
  governs **new** code and code you're **already** refactoring — a
  stable Python module that works is not a rewrite target merely for
  being Python ([§9.1](#hot-path)). Rewrite it to Rust when you're
  touching it substantially anyway, or when a carve-out below forces
  it.
- **Two carve-outs make Rust mandatory even inside the
  Python-allowed domains:** a **measured hot path**
  ([§9.1](#hot-path)) and a **silent-corruption critical path**
  ([§9.2](#critical-path)). A function needs only one to *require*
  Rust.

<a id="hot-path"></a>
### 9.1 Hot path — measured GIL contention

Rust on hot paths that release the GIL. Per-frame work measured
in milliseconds (image decode, shm publisher, bursty I/O
fetchers, large hashes). The Rust crate exposes a Python binding;
the GIL-releasing nogil section is where the win is.

Don't migrate to Rust for performance without a **measured**
GIL-blocking hot path. RSS bloat, leak suspicion, "Rust feels
safer" — none are measured hot paths. Run a memory probe first;
the tracemalloc top-growers report will name the file:line. At
THAT point decide whether the fix is Python (most likely — pre-
allocate, reuse builders, drop a per-tick allocator) or moving
that specific allocator to Rust.

<a id="critical-path"></a>
### 9.2 Critical path — silent-corruption failure mode

Rust when the function's **failure mode is silent corruption,
not a noisy crash**. Performance can be a non-issue — a critical
function may be a cold path hit once per session. The reason to
move it to Rust is that Python's "silently wrong type" and C's
"silently wrong pointer" become **compile errors** or **panics**
in Rust. The bug class collapses at the language boundary.

What counts as critical-path:

- **SPSC ring headers / lock-free queue primitives.** A torn
  write produces wrong bytes downstream, not an exception — and
  the reader trusts the bytes.
- **Buffer slicing across an FFI boundary.** Strided views,
  out-of-bounds reads, use-after-free. Python `bytes` looks
  fine; the C side silently corrupts memory.
- **Reference-counted resource handles.** A drop that releases
  a shared GPU texture, shm segment, or file descriptor. A
  refcount bug surfaces hours later as "tile cache leaks GPU
  memory," not as a stack trace.
- **Lock-free atomic compare-and-swap loops.** Memory-ordering
  bugs on ARM are silent on x86 and corrupt under load on the
  target. Rust's ordering parameters are required arguments,
  not defaults.
- **Actuator-bound command serialisers.** Byte layout the
  firmware reads field-by-field — a stray field shifts every
  downstream offset, the vehicle accepts the malformed frame as
  a valid command for some other axis.
- **Schema enforcement at trust boundaries.** A capnp / protobuf
  read from an untrusted source where a misread field becomes a
  silent wrong default (see [§7.2](#schema-getattr-trap)). Rust's `Result<T, SchemaError>`
  vs Python's `getattr(...)` is the difference.

Python is the right tool when the failure mode is **noisy** — a
`TypeError` / `KeyError` / `ValueError` fires on the next call,
the operator sees a stack trace, a unit test catches it. Python
is the wrong tool when the failure mode is "looks right, behaves
wrong" — silently dropped events, partial state visible to a
reader, half-written file the next reader trusts.

When you propose a Rust migration, name the specific bug class
you're collapsing. "This serialiser publishes bytes the firmware
trusts; today a wrong field type goes unnoticed until the
vehicle gets a bad command" is a justification. "Could probably
use ownership semantics" is not.

<a id="ffi-boundary"></a>
### 9.3 FFI boundary: validate invariants where `unsafe` starts

Any Rust → Python (or Rust → C) FFI surface that takes a buffer
and calls `slice::from_raw_parts(ptr, len)` is sound only when
the buffer is **C-contiguous**. A strided view (e.g.
`numpy_arr[::2]`) reports the same `len` but the slice spans
unrelated memory in between — silent UB, the worst kind of bug,
the kind that turns into "random crashes once a week" reports.

Validate at the boundary, return a typed error, document the fix:

```rust
fn check_contiguous(buf: &PyBuffer<u8>, role: &str) -> PyResult<()> {
    if !buf.is_c_contiguous() {
        return Err(PyValueError::new_err(format!(
            "{role} buffer is not C-contiguous (strided views are \
             unsupported; pass np.ascontiguousarray(...) on the \
             Python side)"
        )));
    }
    Ok(())
}
```

The error message tells the caller exactly what to do. UB on a
silent strided buffer would have produced a memory-corruption
report; an explicit `ValueError` produces a one-line fix.

### 9.4 Name reader vs writer access paths distinguishably

In a shared-memory region used for SPSC publish/consume, the
writer and reader codepaths share the underlying mmap. Give them
**different method names** so a code-review skim can tell which
direction a call site is operating in:

```rust
impl ShmRegion {
    // Writer path — pointer methods named with _mut.
    unsafe fn header_mut(&self) -> *mut u8 { ... }
    unsafe fn pixels_mut(&self) -> *mut u8 { ... }

    // Reader path — explicit _ptr suffix; same address,
    // const-correct intent.
    unsafe fn header_ptr(&self) -> *const u8 { ... }
    unsafe fn pixels_ptr(&self) -> *const u8 { ... }
}
```

The bytes are identical; the names exist so the reader API
**never accidentally calls a method named with `_mut`** in a
codepath that should be read-only. Names are part of the safety
contract, not just style.

<a id="accelerators"></a>
### 9.5 Accelerators (GPU / NPU / DLA): contention, hangs, hardware paths

A GPU / NPU / hardware codec is a **shared, finite, failure-prone
resource** — not "free speed." On an embedded board (Jetson/Orin,
phone-class SoC) the accelerator and its memory are shared with
everything else and run near the edge of the budget. Treat it
with the same discipline as a CAN bus or a single CPU core.

**1. One accelerator, one heavy user per process — don't make two
contexts fight.** Two CUDA/OpenCL contexts in the *same* process
contending for the same device per frame (e.g. a CL preprocess +
a separate inference runtime, both CUDA) serialise and stall —
the model "loads" but never produces output. If a pipeline needs
two accelerator stages, **isolate them** (separate processes, or
one unified context), or keep the contended stage on CPU. Adding
a second GPU user to a process that already has one is a
regression even when each works alone.

**2. Prefer the hardware path over the software one for codec /
fixed-function work.** If the SoC has a hardware encoder/decoder
(NVENC/NVDEC, V4L2 M2M, VideoToolbox), use it — software
`libx264` / CPU decode on an embedded target burns cores you
don't have. (Pixel ops with no fixed-function unit — a
`warpPerspective`, a colour convert — stay CPU unless you can
measure a GPU win; moving them to GPU adds a context, see rule 1.)

**3. A GPU hang looks like a hang, not a crash — and the
supervisor won't catch it.** A driver fence that never signals
leaves the process **alive but blocked**, so a process-liveness
supervisor (§6) sees it as healthy and never restarts it. The
symptom is "everything downstream of the model went silent" with
the process showing multi-hour uptime. Diagnose at the kernel,
not the app:

```sh
cat /proc/$(pgrep -f <inference-proc>)/wchan
# → "dma_fence_default_wait" = blocked on a GPU fence that never
#   signalled. Restart the service; nothing downstream is the bug.
```

This is the accelerator instance of "looks healthy ≠ working"
([anti-patterns recurring shape #2](anti-patterns.md#recurring-shapes)):
assert on output produced, not on the process being alive. A
heartbeat on the accelerator's *output topic* catches it; a PID
check doesn't.

**4. Under memory pressure the driver fails in ways clean code
doesn't.** Near the memory ceiling, a GPU driver may fail to
retire a fence (rule 3) instead of returning an allocation error
you can handle. You can't code around a wedged driver — so keep
headroom: bound accelerator memory (model size, batch, texture
cache — see [§8.8](ui-rules.md#texture-lifecycle)), and treat "the board
sits at 93 % memory" as the root cause to fix, not a backdrop to
debug against.

**5. The accelerator artifact is platform-specific too.** Compiled
kernels, ONNX/TensorRT engines, baked model blobs are built for a
specific driver / arch / compute capability — they're the
[§11.1](delivery-rules.md#platform-libs) rule applied to GPU artifacts: build them
on/for the target, select by platform, regenerate on the rig
(don't commit one platform's engine as "the" engine), and verify
they load before trusting them.

**6. "Loaded" is not "running on the device you think." Report the
active device; don't trust the init message.** Two silent traps,
both about *messages lying about where compute landed*:

- **A "DEFAULT" device selector silently picks the wrong one.**
  `CL_DEVICE_TYPE_DEFAULT` (or any "auto" backend pick) can
  resolve to the **CPU** device on a runtime that exposes both
  CPU and GPU — your "GPU" kernels quietly run on CPU at many
  times the cost, and nothing says so. Select the device
  **explicitly** (`CL_DEVICE_TYPE_GPU`, a named CUDA device, an
  explicit backend), and **log which device actually bound** at
  startup, with its name — not "models loaded."
- **"Models loaded in 1.0 s" tells you nothing about throughput.**
  A model that loads cleanly and then publishes **0 Hz** (a
  context-contention stall, rule 1; a wedged fence, rule 3) looks
  identical at the load message to one running fine. The init log
  is not a liveness signal. Gate on **output rate** (heartbeat
  the model's output topic, [§16.4](delivery-rules.md#live-field-convention)), and
  make the active compute path announce itself —
  `"tinygrad-CUDA preprocessing active — OpenCL bypassed"` /
  `"CPU fallback: no GPU device"`. A device-selection change you
  can't confirm from a runtime message is a change you haven't
  verified. (Accelerator instance of "looks healthy ≠ working,"
  [recurring shape #2](anti-patterns.md#recurring-shapes); same
  loud-default discipline as [§9.6](#build-backend-switch).)

**7. A cached / JIT-compiled path may reuse its output buffer
across calls — clone before you retain it.** A compile-once,
replay-many accelerator path (a JIT'd kernel, a graph-captured
pipeline) often writes into the *same* output buffer every call,
on purpose, for speed. If you keep references across calls — a
temporal ring, a "previous frame" pair, a sliding history window —
successive entries **alias to one buffer**, so "current" and
"previous" read identical or corrupt. `.clone()` (deep-copy) the
result before retaining it. The tell is garbled/duplicated
*history* while a one-shot read looks perfect — so a single-frame
test passes and the bug only shows when something compares across
time.

The generalisable rule: **an accelerator is a contended,
hang-prone, platform-specific resource whose device binding is
silent unless you make it loud. Give it one heavy user per
context, prefer its hardware fixed-function path, select the
device explicitly and log which one bound, detect hangs/stalls on
output rate (not the "loaded" message or PID), keep memory
headroom so the driver returns errors instead of wedging, and
clone a cached path's reused output before retaining it.** See
anti-patterns #38 and #40.

<a id="build-backend-switch"></a>
### 9.6 A build-time backend switch (`CUDA=1`) is a build input — make it explicit, recorded, and verified

A build-time env var that selects a backend — `CUDA=1 scons`,
`USE_GPU=1 make`, `ACCEL=metal cargo build` — is **not a runtime
toggle.** It bakes a *different artifact*: a GPU build and a CPU
build come out byte-different but look identical on disk, and
nothing about the finished binary says which one you got. That's
the trap. The same env var that's harmless for a flag (§7.5) is a
silent-divergence generator when it forks the build, because:

- **It drifts across build hosts.** Rig A's shell has `CUDA=1`,
  rig B's doesn't → two different binaries from one source tree,
  with no record of which is which. The
  [authoritative-vs-actual cycle](anti-patterns.md#recurring-shapes)
  (#1) in build form.
- **Unset silently falls back.** No `CUDA` in the environment
  doesn't error — SCons quietly builds the CPU path. A rig you
  *meant* to be GPU-accelerated ships CPU-only and nobody is
  told. (Silent-falsy cycle, #3.)
- **The cache lies.** SCons keys its build cache on declared
  dependencies; a bare `os.environ["CUDA"]` read is invisible to
  that graph, so flipping the var may **not** trigger a rebuild —
  you get the *other* backend's stale objects. (Cache-vs-code
  cycle, #5 / anti-pattern #26.)

Make the backend selection robust:

- **Promote it to a declared build variable, not a bare
  `os.environ` read.** SCons `Variables()` / `ARGUMENTS`
  (`scons gpu=1`), CMake `-DUSE_CUDA=ON`, a Cargo feature — a
  *first-class* input the build system records and cache-keys on,
  so changing it forces the correct rebuild. If you must read an
  env var, fold it into the build signature
  (`env.Append(... )` / an explicit cache-key value) so the cache
  can't serve the wrong backend.
- **Default explicitly, and make the default loud.** Pick the
  default deliberately (CPU for portability, or detect-and-assert)
  and **log the chosen backend at build time** — `>>> building
  with CUDA=ON` / `CPU fallback (no CUDA toolkit found)`. An
  unset-means-CPU that prints nothing is how a rig silently loses
  its accelerator.
- **Stamp the backend into the artifact** so it's reportable at
  runtime, same as the build stamp (§12.1): `build_version.json`
  carries `"accel": "cuda"` / `"cpu"`. Then "is this the GPU
  build?" is a command (`<app> --version`), not a guess — and a
  GPU rig running the CPU build is caught by reading the stamp,
  not by chasing why inference is slow.
- **Don't let one shared `lib/` hold both backends' objects.**
  A GPU object linked into a CPU build (or vice-versa) is the
  wrong-platform-artifact shape (§11.1) one level in; keep
  per-backend build dirs and select, don't glob.

**The same trap, one axis over: a feature / access tier baked as a
separate build.** A "lite / pro / demo / intern" *build* forks the
artifact for what is really a **runtime role** — two bundles to
build, sign, ship, and keep from drifting, and a user who changes
tier needs a different binary. Ship **one bundle** and gate the
capability at runtime by role / license / login account — not a
build flavor or a `*_BUILD` env (§7.5). One artifact, many roles;
the tier is data the running app reads, not a fork in the build.

The generalisable rule: **a backend-selecting build flag forks
the artifact, so it must be a declared, cache-keyed build input
with an explicit logged default, stamped into the output and
reportable at runtime — never an undeclared `os.environ` read
that drifts between build hosts and falls back in silence.** See
anti-pattern #39.

<a id="cpu-gpu-decision"></a>
### 9.7 Stop the CPU↔GPU oscillation: measure once, record the verdict, make it switchable

A pipeline stage that keeps flip-flopping — "move it to the GPU"
→ slow/0 Hz → "revert to CPU" → "no really, GPU" — is burning
days re-litigating a decision that was never *measured* or
*recorded*. The thrash has three fixable causes; fix all three
and the battle ends.

**1. Measure the real cost — separate device time from harness
overhead.** The numbers that drive a CPU↔GPU flip are usually
wrong. A GPU path benchmarked at "220 ms/frame, too slow" was
**not** GPU time — it was a framework re-tracing its graph in
Python every frame; compiled/cached (a `TinyJit`-style replay)
the *same* path was ~14 ms. Conversely a "GPU" path can be silently
running on CPU (§9.5 r6). Before you switch, profile the stage in
isolation and attribute the time: kernel vs. host-launch vs.
per-frame graph build vs. context thrash. A flip justified by an
un-attributed wall-clock number is a coin toss, not a decision.

**2. Until the faster path is validated, default to what runs;
once it's validated on the target, the validated path is the
default.** *Before* the accelerated path is proven on the real
hardware, the interim default is the one that produces output
today (often CPU) — but mark it **interim** ("GPU pending
validation"), not a verdict. The moment the faster path is
**validated on the target** (correct output, at rate, on the real
device — not a dev box), it becomes the **recorded default**
([§9.10](#validated-accelerator-default)); leaving the default on
CPU "because it's safe" after the GPU path works is how a rig
provisioned for the accelerator quietly ships without it. Either
way the *other* path is a **runtime switch, not a rebuild** (§7.5
config, not a build fork §9.6): if the accelerated path fails to
init or drops below a rate floor, any fall back to CPU is a
**loud, alarmed, temporary degraded mode** — never a silent revert
([§9.10](#validated-accelerator-default)). When switching is a
config toggle, an experiment is one restart, not a
rebuild-redeploy-revert cycle — the oscillation stops being
expensive.

**3. Write the decision down with its numbers and its blocker.**
A reverted GPU attempt that leaves no record gets re-attempted by
the next person from scratch. Record: the measured numbers (with
the attribution from rule 1), *why* it was shelved (e.g. "two
CUDA contexts contend → 0 Hz, §9.5 r1 — needs them on one
stream"), and the precise condition that would make it worth
re-trying. Then a "should we move this to GPU?" is answered by
reading, not by re-running the whole loop. (This is the general
decision-record rule [§18.3](delivery-rules.md#decision-record) — the sixth
recurring shape — applied to a perf decision, like the migration
map §18.2.)

**Is the win even worth it?** A GPU port that saves 12 % of one
stage's CPU but adds a second accelerator context (§9.5 r1), a
platform-specific build (§9.6, §11.1), and a new failure mode
(§9.5 r3) is often net-negative on a board that isn't GPU-bound.
Name what's actually saturated first (a CPU profile) — moving a
non-bottleneck stage to the GPU is motion, not progress.

The generalisable rule: **CPU-vs-GPU is decided once, by attributed
measurement, recorded with its blocker; the chosen default is the
*validated* path — the accelerated one on hardware that has the
accelerator ([§9.10](#validated-accelerator-default)); the
alternative is a logged runtime switch with a loud degraded
fallback, never a rebuild war.** See anti-pattern #42.

<a id="device-single-source"></a>
### 9.8 One device resolver for build *and* runtime; the allocation is a written policy

Once §9.7 has settled which stage runs where, **lock it in two
ways** so it can't quietly come undone:

**1. Build and runtime must resolve the device from the *same*
source.** The crash this prevents: the build tagged an artifact
for one device (a SConscript arch-map defaulting to CPU) while the
runtime hardcoded another (`DEV=CUDA` in the loader) — producing a
CPU-tagged model blob that **JIT-mismatched** the runtime and
crash-looped the process (no output → downstream NaNs). Put the
choice in **one resolver** that both the build script and the
runtime import:

```python
# device.py — the single source of truth
def preferred_device():      # build + runtime agree by construction
    if is_jetson(): return "CUDA"      # fail-loud on an unmapped target
    ...
def runtime_device():        # obeys the artifact's own sidecar
    return _sidecar_device() or preferred_device()
```

The build forces that device on **every** path (prebuilt-guard
*refuses* a wrong-device artifact; compile path *asserts* it), and
the artifact carries its device in a sidecar the runtime reads
back — so a device mismatch is a loud refusal at load, not a
crash-loop at runtime. (This is §9.6's "stamp the backend into the
artifact" + §11.1's "verify what loaded," made the *single*
authority.)

**2. The per-process CPU/GPU allocation is written down where the
process table lives** (§6) — which daemons are GPU-accelerated
(and on which engine: GPU compute, hardware-codec, NPU) and that
**everything else is CPU on purpose.** State the rule that a
CPU/POCL fallback for a daemon that's *meant* to be GPU is a
**regression to fix at its source, not a runtime knob.** Written
into the process config, the policy is unmissable and can't
silently revert to the slow path — the documented allocation is
the thing #42's oscillation kept failing to have. The validated
accelerated path is that recorded default
([§9.10](#validated-accelerator-default)).

The generalisable rule: **resolve the compute device in one place
both build and runtime obey (mismatch = loud refusal, not a
crash-loop), and record the per-process GPU/CPU allocation as
policy next to the process table — a fallback to CPU for a
GPU-designated stage is a bug, not a setting.** See
anti-pattern #43.

<a id="shared-resource-contention"></a>
### 9.9 A deliberate resource cap can be load-bearing; CPU priority won't isolate bandwidth contention

On a board where the compute units share one memory controller —
a Jetson/Orin's unified memory, a phone-class SoC, any CPU+GPU on
one bus — the resource that's actually scarce is often **invisible
to the obvious tool.** Two failure modes recur, and both bite when
you "optimise" by reasoning about CPU cores alone.

**1. A throughput cap that looks like a bug can be load-bearing —
find what it protects before you "fix" it.** A core pin that
serialises six capture threads onto one core while five sit idle,
a rate limit that "wastes" headroom, a deliberately small batch —
these read as obvious inefficiencies. Some are deliberate
**throttles that cap one process's resource draw to leave headroom
for a latency-critical peer.** Widening such a cap to "use the idle
capacity" raises the throttled process's throughput and *collapses
the protected one*: we widened a capture daemon's core pin from one
core to five — capture frame-rate rose, the safety-critical
inference model's rate fell by half, and the board watchdog-rebooted
under the load spike. Re-pinning live recovered the model with
nothing else changed — proof the cap was the lever, not a bug.
Before you touch an apparent inefficiency, ask **what consumer it
protects**, and **measure the change against that consumer's output
rate** — not the throughput of the thing you sped up. A win on the
optimised stage that regresses the protected peer is a regression.

**2. CPU priority and affinity do NOT isolate memory-bandwidth /
GPU / I/O contention.** `nice -19` a heavy build and a
latency-critical model still starves — they contend for the
**memory bus**, not CPU time, and the scheduler doesn't arbitrate
the bus. "It's reniced / on a separate core / in a separate build
dir" does **not** mean "it won't interfere": separate dir, same
bus. The levers that *do* work either reduce the contender's draw
on the shared resource (cap its throughput — rule 1; move fewer
bytes; rate-limit *before* the expensive stage, not after) or
**don't co-locate heavy work with the latency-critical process at
all** — stop the service for a maintenance window, build there,
restart.

**A shared network uplink is the same contended resource.** Running
*more* large transfers in parallel over a thin or contended link is
net-negative: N concurrent multi-MB PUTs each get ~1/N of the
bandwidth, so they all crawl, blow a fixed request timeout, and
**502 / time out together** — while serial (or low-concurrency)
transfers complete. We shipped this: an uploader at `PARALLEL=3`
sending 9 MB blobs over an uplink *shared with live teleop video*
lost almost every large blob to read-timeouts, while a single PUT
of the same file succeeded. Two fixes, both general: **serialize or
adapt the concurrency** (and treat the cap as load-bearing — it
also protects the latency-critical co-traffic, here teleop), and
**size the timeout to `bytes ÷ worst-case bandwidth`, not a
constant** — a 10 s timeout tuned on a fast link guarantees failure
for a streamed multi-MB body over a slow one.

**Annotate load-bearing constraints in place.** A core pin, a rate
cap, or a batch limit that protects a peer must carry an inline
comment saying *what it protects and what breaks if you widen it* —
the [§3 safety-watchdog "threshold documented in place"](#safety-critical)
and [§1 load-bearing-name](#naming) discipline applied to a resource
budget. An unexplained constraint reads as an accident and gets
"optimised" away by the next reader (or the next AI).

The generalisable rule: **on a shared-memory board, throughput
trades against a latency-critical peer through a resource the
obvious metric doesn't show; a constraint that throttles one
process may be protecting another, so find what it protects and
measure the *protected* consumer before changing it — and reach for
reduce-the-draw or don't-co-locate, never `nice`, to win back
bandwidth.** See anti-pattern #44.

<a id="validated-accelerator-default"></a>
### 9.10 Once validated on the target, the accelerated path is the default — a revert to CPU is a regression, not a fallback

Hardware is *provisioned for* its accelerator: a GPU SoC, an NPU
board, a fixed-function codec exists so the heavy path runs there.
On that hardware the **intended steady state is the accelerated
path** — CPU is the bring-up scaffold and the degraded mode, not
the destination. §9.7 stops the oscillation by making the decision
measured, recorded, and switchable; this rule fixes the part §9.7's
"default to what runs" leaves open — *which way the settled default
points* — so the pendulum doesn't keep swinging back to CPU.

**A decision *validated on the target* is settled.** Once the
accelerated path has been shown to run correctly, at rate, on the
real hardware — not a dev box, the target — that result is the
**recorded default** ([§9.8](#device-single-source) allocation
policy) and the question is closed. Re-opening it takes the *same
evidence bar* that opens any settled decision: a new **attributed**
measurement ([§9.7 r1](#cpu-gpu-decision)) that overturns the
recorded result — never "it felt slow," an un-attributed number
([§15.6](delivery-rules.md#optimization-floor)), or a fresh engineer's unease. "Let's
just try CPU again" against a validated GPU path is re-litigation,
and it's how the war restarts.

**A CPU fallback is a *loud, temporary, degraded* mode — never the
silent steady state.** If the accelerated path fails to init or
drops below its rate floor, falling back to CPU is acceptable
*only* when it is alarmed and surfaced (an operator-visible degraded
state, a logged regression to fix). For a stage the hardware was
provisioned for — especially a safety- or latency-critical one —
running silently at a fraction of the intended rate is often
**worse than refusing to start**: a loud refusal gets fixed, a
silent 15×-slower path ships. A silent CPU fallback nobody notices
is the [§9.8](#device-single-source) "regression, not a setting"
shipping itself.

**Direction matters in the default.** §9.7 r2's "default to the
path that runs" is the posture *during bring-up, before
validation* — marked as interim, not a verdict. The moment the
accelerated path is validated, the default flips to it; treating
CPU as the permanent "safe" home after the GPU path works is how a
rig quietly runs without the accelerator it was built around.

The generalisable rule: **on hardware provisioned for an
accelerator, the validated accelerated path is the recorded
default; CPU is bring-up scaffold or a loud degraded mode, not the
steady state; and a validated GPU/CPU decision is re-opened only by
new attributed measurement that overturns the record — not by
unease.** See anti-pattern #42.

---

<a id="per-stream-isolation"></a>
## 10. Per-stream isolation

> **When this fires** — media streams, per-camera / per-device workers, or a shared diagnostic probe.
>
> **Do** — isolate per stream so one stall can't freeze the rest; a shared probe must not false-flag an encode-only source.

Independent media streams (each camera, each microphone, each
sensor pipeline) run in **independent OS processes** wherever the
transport allows. A single decoder hang or buffer corruption must
not be able to freeze the other streams.

The shared-memory frame buffer is the common API across transports:
all stream workers — direct-IPC, network, replay — write into the
same buffer shape so the consumer doesn't care which transport
produced the frame. Don't collapse stream workers into the consumer
process "to save overhead."

Per-process input that has a non-trivial chance of segfaulting on
specific hardware adversity (HID libraries, vendor SDKs) belongs
in its own subprocess for the same reason — its crash doesn't
take the consumer down. Run an orphan sweep at startup to clean
up any children left behind by a prior ungraceful exit (see [§8.11 orphan sweep](ui-rules.md#orphan-sweep)).

<a id="shared-probes"></a>
### 10.1 Shared probes, not per-consumer probes (best-path tracker)

When a system has **multiple parallel transports** between two
endpoints (LAN-direct vs cloud-tunnel, primary vs failover, fast
vs slow), don't let each consumer probe independently. Run **one
background probe** at a low frequency, publish **one snapshot**,
have every consumer poll the snapshot on its next operation.

```python
@dataclass(frozen=True)
class PathSnap(FrozenSnapshot):
    via: str           # "lan" | "tunnel" | "relay"
    rtt_ms: float
    last_check_t: float

class PathTracker(SnapshotSource[PathSnap]):
    """Probes BOTH transports every probe_interval_s, publishes
    the current best route. Consumers (SSH-based services, the
    status banner) poll the snapshot — no per-call probes."""
    ...
```

Three reasons this beats per-consumer probes:

1. **One probe loop.** N consumers × M transports × probe-per-
   call collapses to one probe at a low duty cycle.
2. **Auto-failover.** When LAN drops, the next probe sees it; the
   snapshot flips; every consumer slips onto the surviving path
   on its next operation. Reverse holds when LAN comes back —
   consumers slip back without explicit re-config.
3. **One banner.** Operators see the path change in one place
   (a status banner reading the same snapshot). No log-tail
   spelunking. No "why did this SSH go via the VPS?" — the
   answer is the snapshot field.

**Pick by sustained rate, not mere "alive."** "Best" / "preferred"
has to mean *delivering at rate*, not merely *present*. A preferred
source that's alive but **scheduler-starved or rate-throttled**
(replying, but at a fraction of its target rate) is worse than the
fallback — the consumer shows a frozen/stale view while a boolean
"alive" probe still says fine. Gate the preference on a **rate
floor** the probe measures (`prefer A only if A.hz ≥ floor, else
B`), so a degraded primary self-demotes to the live fallback and
self-promotes when it recovers. (Pick the floor just under the
source's target rate.) This is the
[§8.20 alive-but-silent vs broken](ui-rules.md#overlay-staleness) distinction
applied to source *selection*: aliveness is not liveness, and a
selection predicate that gates on "alive" picks the stale source.

Structurally the same shape as the snapshot pattern in [§8.2 snapshot pattern](ui-rules.md#snapshot-pattern),
applied to **connection state** instead of **application state**.
Same producer rules: build the snapshot complete, swap under
one lock, immutable consumers, `live: bool` field on snapshots
that may legitimately stop being updated (network down, all
candidates timed out).

<a id="capped-recovery"></a>
### 10.2 Capped auto-recovery for a single degraded element

When one element of an N-element system degrades (one of six
camera streams stalls while the other five are fine), the system
should **auto-recover that one element without operator action**
— but the recovery must be **capped** so a permanently-broken
element doesn't thrash forever.

The shape:

```python
# Per-element stall watchdog. If a stream's last-frame age
# exceeds the threshold, bounce ITS connection — not the whole
# transport. Cap the bounces so a dead camera doesn't loop.
def _check_per_track_stalls(self):
    for el in self.elements:
        if el.age() > STALL_TIMEOUT_S and el.bounces < MAX_BOUNCES:
            el.bounce()                 # targeted recovery
            el.bounces += 1
            log.warning("%s stalled %.1fs — bounce %d/%d",
                        el.name, el.age(), el.bounces, MAX_BOUNCES)
        # after MAX_BOUNCES: leave it dead, surface as stale in UI
```

Four rules:

1. **Recover the element, not the system.** Bouncing the whole
   transport because one stream stalled punishes the five healthy
   streams. Target the broken one.
2. **Cap the recovery.** `MAX_BOUNCES` (or a rate limit) stops a
   genuinely-dead element from a thrash loop that burns CPU and
   floods the log. After the cap, leave it degraded and surface
   the staleness in the UI ([§8.20 overlay staleness](ui-rules.md#overlay-staleness)).
3. **Log each recovery at WARNING with the count.** `bounce
   3/5` tells the operator (and the assistant) the element is
   struggling, before it hits the cap and goes dark.
4. **The degraded-signal must encode each element's *expected*
   state — or it false-flags a by-design difference and "recovers"
   a healthy element.** If elements are configured differently on
   purpose (4 cameras encode-only, 2 raw), a probe that measures
   one metric uniformly (raw-frame rate) reports the encode-only
   ones as "dead" *every time* — a permanent false positive
   ([always-red, §16.6](delivery-rules.md#red-is-no-signal)). That's noise at best;
   at worst it **triggers a destructive recovery on a working
   element** — here a false "camera degraded" drove a camera-reset
   that tripped a whole-rig reboot. Gate recovery on the element's
   *own* expected channel (encode-data Hz for an encode-only cam,
   raw-VIPC Hz for a raw one), and verify it is genuinely down on
   that channel before any reset ([capture before reset,
   §15.7](delivery-rules.md#no-reflexive-reboot)).

**Detect per element — an aggregate health metric hides a single
dead one.** The reason you need a per-element watchdog at all: a
*global* health signal (total frames across all streams, total
throughput, "≥1 consumer draining") stays green while one element
is wedged, because the healthy elements keep the aggregate moving.
A single camera can sit stale for minutes behind a "frames are
flowing" metric that only ever summed all cameras. Liveness must
be tracked **per element** (per stream's last-decode time, per
consumer's last-progress time), not as a sum — the sum is exactly
what masks the failure. See anti-pattern #33.

The auto-recovery path must be the one that **actually runs** —
see anti-pattern #23 (a recovery mechanism that's dead code while
the operator believes it's protecting them; e.g. an RTP keyframe
watchdog on a DataChannel-only transport never fires).

---

## 11. Cross-compile

> **When this fires** — building or freezing for a different architecture / target.
>
> **Do** — cross-build with `zigbuild`; install every native dep in the build env; verify the linked arch and by importing entry points — not the exit code.

**Full text moved to [`delivery-rules.md` §11](delivery-rules.md#cross-compile).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 12. Syncing source to an embedded target

> **When this fires** — rsyncing source to an embedded target, or deploying across many rigs.
>
> **Do** — exclude native-arch binaries from the sync; stamp `git_sha`+dirty and diff stamps before debugging; gate compatibility on feature-set ⊆ advertised, not a version string.

**Full text moved to [`delivery-rules.md` §12](delivery-rules.md#embedded-sync).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 13. Source tree vs. deployed artifact

> **When this fires** — producing build artifacts, consuming a shared library, or running a path on a compiled / installed rig.
>
> **Do** — gitignore generated output; depend on a **published version** (never a sibling source tree); don't assume the dev layout at runtime; a hot-edit on the deployed box is a *loan* — capture it back same-day.

**Full text moved to [`delivery-rules.md` §13](delivery-rules.md#source-vs-deployed).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 14. Bundled configuration and shipping defaults

> **When this fires** — shipping configuration inside a frozen / deployed bundle.
>
> **Do** — bake explicitly (never automatically), gitignore the output, and have the runtime **seed from it, not trust it**.

**Full text moved to [`delivery-rules.md` §14](delivery-rules.md#bundled-config).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 15. Diagnostics

> **When this fires** — chasing a bug, about to reboot to clear a symptom, or about to call something a performance 'floor.'
>
> **Do** — **diagnostic-first** — produce the command that proves the cause before you change code; capture volatile state before any reset; name the mechanism and the alternatives ruled out.

**Full text moved to [`delivery-rules.md` §15](delivery-rules.md#diagnostics).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 16. Logging discipline

> **When this fires** — logging inside a loop / hot path, or on an embedded kernel console / serial UART.
>
> **Do** — transition + heartbeat, **never per-tick**; rate-limit per source — a slow sink pins a CPU core and can reboot the SoC.

**Full text moved to [`delivery-rules.md` §16](delivery-rules.md#logging).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 17. Validation surfaces

> **When this fires** — a workflow has a canonical entry point (an onboarding dialog, a bootstrap script, a first-enroll flow).
>
> **Do** — fix the **canonical path**; a side-channel that works while the canonical path fails *is* the bug, not a workaround to ship.

**Full text moved to [`delivery-rules.md` §17](delivery-rules.md#validation-surfaces).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 18. Tests / verification

> **When this fires** — writing or fixing code with a testable contract, or about to claim 'done.'
>
> **Do** — **test-first** (RED → GREEN → REFACTOR); state what you ran and what it told you; a wrapper propagates the real exit code; settle a decision with recorded evidence.

**Full text moved to [`delivery-rules.md` §18](delivery-rules.md#tests).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 19. Reversibility

> **When this fires** — a destructive op, a secret, a push, or a delete / cleanup that selects targets by a glob.
>
> **Do** — **ask first** for that specific destructive action; delete an explicit allowlist, never a wildcard; enforce a hard cap with a deterministic, self-contained key.

**Full text moved to [`delivery-rules.md` §19](delivery-rules.md#reversibility).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 20. Repo hygiene

> **When this fires** — pushing, merging to main, or doing concurrent / risky work on a shared, live checkout.
>
> **Do** — right remote and branch; fast-forward only your **committed + validated** branch (never `git add -A`); isolate parallel work in its own **git worktree**.

**Full text moved to [`delivery-rules.md` §20](delivery-rules.md#repo-hygiene).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 21. Python environment management

> **When this fires** — adding or changing a Python dependency.
>
> **Do** — `uv add` and commit `uv.lock` — **never** a bare `pip install`.

**Full text moved to [`delivery-rules.md` §21](delivery-rules.md#python-env).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 22. When you add a new daemon, service, or topic

> **When this fires** — adding a new daemon, service, or topic.
>
> **Do** — give it a name, a consumer, a run-gate, and a written rationale — all four, or it isn't ready.

**Full text moved to [`delivery-rules.md` §22](delivery-rules.md#new-service-checklist).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 23. Network topology

> **When this fires** — a network edge / tunnel / firewall, a web app behind a reverse-proxy subpath, or reaching a fleet host by IP.
>
> **Do** — the cloud edge is plumbing (reverse tunnel for off-LAN); relative URLs / one base path through the proxy; ask the registry's heartbeat-reported `lan_ip`, never a hardcoded IP or a subnet scan.

**Full text moved to [`delivery-rules.md` §23](delivery-rules.md#network-topology).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 24. When in doubt

> **When this fires** — the change crosses repos you can't see, a constraint is unwritten, or two non-obvious paths exist.
>
> **Do** — **stop and ask** — a clarifying question is cheaper than a wrong guess.

**Full text moved to [`delivery-rules.md` §24](delivery-rules.md#when-in-doubt).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 25. Integrating with an external service / API

> **When this fires** — a two-way sync, any write to an external service, an event / webhook integration, or OCR / LLM field extraction.
>
> **Do** — a periodic **reconcile poll** is the correctness floor (push is latency only); idempotent upserts keyed on a stable id; read the most structured source, validate the lossy one.

**Full text moved to [`delivery-rules.md` §25](delivery-rules.md#external-integration).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 26. Handling money or privileged actions: the server is the trust boundary

> **When this fires** — a money / authorization value arrives from the client, a high-impact irreversible action, or a public / no-login demo.
>
> **Do** — recompute money & authz server-side from authoritative data; **dual control** + an audit trail for irreversible actions; isolate a demo from prod by construction.

**Full text moved to [`delivery-rules.md` §26](delivery-rules.md#trust-boundary).** The number and every slug anchor are unchanged, so existing references still resolve.

---

## 27. Fixing a build systematically

> **When this fires** — a build is failing, behaving differently across hosts, or green-but-wrong.
>
> **Do** — method, not flag-poking: reproduce from a clean committed tree → pin every input → localise the layer → build for the target → verify on the **deployed** form.

**Full text moved to [`delivery-rules.md` §27](delivery-rules.md#build-fix).** The number and every slug anchor are unchanged, so existing references still resolve.

---

<a id="reuse-critical-path"></a>
## 28. Reuse the critical path — one module, not many copies

> **When this fires** — reusing or reimplementing critical-path logic (a decoder, transform, safety gate, codec), or fixing shared code.
>
> **Do** — one shared, validated module every caller imports — cross-project that home is **`unolib`**; and a shared-code fix is done at **rollout**, not at compile.

A critical path — a sensor decoder, a coordinate transform, a safety
gate, a wire codec, a controller — must live in **one shared,
validated module that every caller reuses.** A second copy of
critical logic is the most expensive duplication there is: the two
drift, the validated one and the fork disagree, and the divergence
is **silent** until the path that matters produces a wrong number.

- **Reuse-first is the default reflex — reach for the shared module
  before you write the logic.** And this isn't only for the *critical*
  path. Before hand-rolling connection/transport, path-selection,
  config-parse, retry/backoff — *any* logic a second caller will plausibly
  need — first check whether a shared module already does it and wire to
  it. The validated module is fewer lines and carries every bug it already
  fixed; a fresh copy silently re-earns all of them (the LAN-first SSH
  path-pick and the muxed-transport modules each exist because three
  callers were about to fork the same connection logic). Writing a one-off
  where a module exists is the thing to justify — reuse is the default.
- **Reuse the proven implementation; don't reimplement it.** When a
  new feature needs critical logic that already exists and works
  (a radar decoder feeding a *new* control path), wire to the
  **existing** one — don't write a second decoder "just for this." A
  reimplementation has to be re-validated from scratch and drifts
  the moment either copy changes; the reused one carries its
  validation with it.
- **Factor it out the moment a second caller appears.** Logic that
  was fine inline in one `__main__` becomes a **shared module** the
  instant a second consumer needs it — extract it (a `decoder.py`
  lifted out of the daemon), give it one home, have both callers
  import it. Two callers of one function can't diverge; two copies
  of one function always will. (This is
  [§13](delivery-rules.md#source-vs-deployed)'s single-source-of-truth applied
  *within* a codebase, and the inverse of the migration-drop trap
  [§18.2](delivery-rules.md#migration-completeness).)
- **Cross-project reusable code lives in `unolib` — the org's shared
  library, Rust-first.** When the logic is useful beyond one repo (a
  decoder, transform, codec, client, safety gate), its one home is
  **`unolib`**, written in Rust where practical
  ([§9](#rust-vs-python)) so every project gets the same fast,
  validated implementation by depending on a **published version**
  ([§13](delivery-rules.md#source-vs-deployed)) — never a copy pasted into each repo.
  Make it a reflex: *"is this already in `unolib`?"* before you write
  it, and *"should this go **into** `unolib`?"* the moment a second
  project needs it.
- **The topology — every repo is standalone and compiles INTO `unolib`;
  the system runs on `unolib`.** Each repo (`unomsg-rs`, `unolocalization`,
  `unostream`, a stack repo, …) is a **standalone** source repo that builds
  and **publishes its crates/libs into the shared `unolib` prefix**
  ([§13.2](delivery-rules.md#depend-on-prefix)); every consumer depends on
  **`unolib`** (the published bare repos / `lib`+`include`, pinned by tag) —
  never on another repo's source tree, and never by running another repo's
  *daemon* inside your stack. So **"standalone" means a standalone repo that
  publishes to `unolib`, NOT "never reused."** The way to make the system
  more powerful is to **improve the published crate at its source repo,
  republish, and bump consumers** ([§28.1](#roll-out-shared-fix)) — e.g. a
  stack's localizer reuses `unolocalization`'s `unoloc-ekf` / `unoloc-lidar`
  **via `unolib`**, rather than forking the ESKF math or wiring the foreign
  daemon in. Reusing another repo's published *crates* is the design;
  depending on its source tree, or running its daemon inside your stack, is
  not.
- **One implementation is where the hardening lives.** The shared
  module is the single place to put the validation
  ([§3.5](#validate-safety-input)), the Rust rewrite for a
  silent-corruption path ([§9.2](#critical-path)), the fail-safe —
  and every caller inherits it. A forked copy inherits none of it.
- **A critical fork is loud and temporary, never silent.** If you
  genuinely must branch a critical path (a migration, an
  experiment), it's a **recorded decision** ([§18.3](delivery-rules.md#decision-record))
  with a plan to reconverge — not two copies that quietly live on.

The generalisable rule: **critical-path logic lives in one shared,
validated module that every caller reuses — factor it out when the
second caller appears, never reimplement it; two copies of critical
code drift silently and the wrong one eventually drives.** See
anti-pattern #69.

<a id="roll-out-shared-fix"></a>
### 28.1 A shared-code fix is done at *rollout*, not at compile — version the lib, propagate with one tool

Putting the logic in `unolib` is only half the win. A fix to a shared
library reaches a consumer **only when that consumer adopts the new
version** — so "I fixed it and it compiles here" has *not* landed it in
the fleet. The change-only-helps-the-repo-I'm-in pain is the same shape
as [#69](anti-patterns.md#recurring-shapes) one level up: the fix
exists in one place and never propagates. The cure is to make
*rollout* part of "done", with a model that **scales to many repos
instead of fighting it** — and you already run it for these rules
(submodule + [`bump.sh`](delivery-rules.md#source-vs-deployed) / daily-bump + the
`check.py` gate):

- **One home, consumers depend by published version** ([§28](#reuse-critical-path),
  [§13.2](delivery-rules.md#depend-on-prefix)) — `unolib` is published; each repo pins a
  tag/version. Never a pasted copy (that's the unmaintainable path).
- **Every publish ships every architecture the fleet runs — the
  `unolib` floor is linux x86_64 *and* aarch64, both built and
  validated.** The consumers span both (arm64 embedded rigs, x64
  servers and dev boxes), so a single-arch publish silently strands
  half the fleet: their pin can't advance, or — worse — a
  wrong-platform artifact slips through
  ([anti-patterns #4/#36](anti-patterns.md#recurring-shapes)). Build
  the missing arch with the [§11](delivery-rules.md#cross-compile) cross-compile path
  (`zigbuild`), lay artifacts out per `(os, arch)` so consumers select
  by platform (§11.1), and verify each by loading it — not by the
  build exiting 0 ([§18.4](delivery-rules.md#propagate-exit-code)). A version that
  doesn't exist for an architecture is a version that arch can never
  roll to.
- **Fix once, then roll out with one command.** Publish the new
  `unolib` version, then a **fleet bump** advances the pinned version
  in *every* consumer (the code analog of `bump.sh --all` for the
  rules) — you edit one validated implementation and propagate
  mechanically, not by hand-patching N repos.
- **Pin by default; don't "float" to auto-propagate.** A floating
  dep (track `main`/latest) makes fixes spread automatically but
  re-introduces the silent-breakage §13 pinning exists to prevent — a
  consumer silently gets a change it never reviewed. **Pin + a tooled
  rollout** gives both reach *and* per-consumer safety; that pairing is
  what keeps a large multi-repo *more* maintainable than copies, not
  less.
- **Verify the fix actually arrived.** A version stamp
  ([§12.1](delivery-rules.md#build-version-stamp)) lets you diff which consumers carry
  the fix — "published" is not "rolled out", and "rolled out" is not
  "running" ("deployed" ≠ applied, [§3.5](#validate-safety-input)).
  A hotfix that must span everything *now*: publish → fleet-bump
  `--all` → confirm by stamp.

The generalisable rule: **a shared-code fix isn't done when the copy in
front of you compiles — it's done when it's published and rolled out to
every consumer; keep the lib pinned per consumer for safety but own a
one-command fleet rollout plus a staleness/version gate for reach. That
combination — not N hand-patched copies, and not a silent floating dep
— is what lets a fix span the whole codebase and still keeps a large
multi-repo maintainable.** See anti-pattern #69.

<a id="cross-language-lib"></a>
### 28.2 A cross-language shared lib publishes a stable narrow ABI + a thin client — consumers own the schema

When a shared library is consumed across **languages or many repos**
(the Rust `unolib` libs — `unomsg`, `unostream`, `unolocalization` —
consumed from Rust / Python / C++), don't export the lib's internal
types and force every consumer to recompile against them. Publish a
**stable, narrow C ABI** plus a **thin client crate** that links the
compiled per-triple dylib ([§11.4 relocatable rpath](delivery-rules.md#relocatable-rpath),
[§13.2 the prefix](delivery-rules.md#depend-on-prefix)), and keep the **schema on the
consumer side**:

- **The client is schema-agnostic; each consumer owns its schema.** The
  client is generic over the wire type (`CapnpPublisher<T>`); every
  consumer compiles its *own* schema (its own capnp), vendored with a
  **file-id / lineage guard** ([§28 sync-schema](#reuse-critical-path),
  [§2.1](#timestamp-contract)) so copies stay in lineage without the lib
  coupling to any one consumer's wire — and consumers don't couple to
  each other.
- **The ABI is the compatibility contract.** An **additive** ABI change
  (new exports) keeps every existing consumer linking the new dylib with
  no reship. A change to the **wire format / shared-memory header /
  on-disk layout** breaks old binaries — that one must **reship every
  triple and re-pin every consumer** ([§28.1](#roll-out-shared-fix),
  [§4 append-only schemas](#schemas)). Know which kind your change is
  before you publish; the dylib is shared (one per triple, not
  version-namespaced), so a format bump silently breaks consumers that
  merely *restart*.
- **Ship the published surface, not the internal workspace.** A lib
  often splits into an internal `crates/` workspace **and** the thin
  `client/` that consumers actually depend on (the publish mirrors only
  `client/`). Editing the workspace alone **does not reach consumers** —
  change the published client subtree, bump its version, re-publish.
  Verify by building a *consumer* against the new prefix, not by the
  lib's own tests ([§13.2](delivery-rules.md#depend-on-prefix), [§18.7](delivery-rules.md#finish-the-operation)).
- **Don't resolve config relative to the CWD.** If the client finds its
  service/registry table via `./config/…`, a consumer launched outside
  its repo root silently falls back to the dylib's *embedded* table and
  fails (`UnknownService`). Let the consumer **register its table
  explicitly** at init (or embed it), so resolution is independent of
  where the process was started ([§7](#silent-defaults),
  [§13.1](delivery-rules.md#dev-vs-deployed-layout)).

The generalisable rule: **a library reused across languages / repos
exposes a stable narrow ABI and a thin client, keeps the schema on the
consumer side (each owns its own, lineage-guarded), and treats the ABI as
the contract — additive changes need no reship, a wire/format/header bump
reships every triple and re-pins every consumer; ship the published
client surface (not the internal workspace), and don't resolve config
from the CWD.** See anti-pattern #101.

<a id="lib-version-stamp"></a>
### 28.3 A built lib carries a queryable version — verify the one that's *loaded*, and rebuild every triple together

Built libraries are **version-mixup machines**: one source, N per-triple
artifacts, a shared prefix, and consumers that pin. The moment a rebuild
is **partial** — one triple rebuilt and not the others, a fresh mac
dylib over stale linux ones, a re-publish that updated the source tag but
not every artifact — the prefix holds **mixed versions** and nobody can
tell, because the source `Cargo.toml` / `pyproject` says one thing while
the bytes on disk say another. "I rebuilt it" is not "the loaded one is
new."

- **The artifact self-reports its version, queryably.** Embed the
  version (and ideally the `git_sha` + dirty flag, [§12.1](delivery-rules.md#build-version-stamp))
  *in the built lib* — an exported `…_version()` symbol, an embedded
  string `strings` can find, or a sidecar manifest next to the `.so`.
  Then "what version is this dylib?" is a command, not a guess.
- **Verify the version that's *loaded / linked*, not the source.** Check
  the artifact in the prefix, and the one the consumer actually resolves
  at runtime (call its `…_version()`, `otool -L`/`ldd` the path, read the
  manifest) — never infer it from the repo you're standing in. This is
  [§13.1](delivery-rules.md#dev-vs-deployed-layout) ("verify on the deployed form") and
  [§28.1](#roll-out-shared-fix) ("confirm by stamp") for the lib itself.
- **Rebuild and publish *all* triples from one version, atomically.** A
  per-triple build that does some arches and not others is the stranding
  in [§28.1](#roll-out-shared-fix) / [§11.3](delivery-rules.md#cross-build-matrix); drive
  the whole matrix from one command and one version so the triples can't
  drift. A shared, non-version-namespaced dylib makes this worse — a half
  rebuild silently gives different consumers different bytes for the same
  pin.
- **One source of the version number.** Read the version from one place
  (the workspace package field) into the artifact stamp, the published
  tag, and the manifest — three hand-kept copies drift (the `client/`
  vs `crates/` split, [§28.2](#cross-language-lib), makes "which one is
  authoritative?" its own trap).

The generalisable rule: **a built lib carries a queryable version stamp
(embedded symbol / string / sidecar, with `git_sha`), you verify the
version actually *loaded or linked* rather than the source you're looking
at, and you rebuild + publish every triple from one version atomically —
partial rebuilds across triples and a shared prefix are the default way
built-lib versions get mixed up, and the source never tells you which
bytes are really there.** See anti-pattern #103.

<a id="single-version-graph"></a>
### 28.4 A shared lib resolves to one version in the dependency graph — align the pins

A shared library that defines **types crossing component boundaries** — a
wire / message type, a `Service` struct, a C ABI — must resolve to
**exactly one version** in any single build's dependency graph. The
moment two consumers (or a transitive dep) pin **different** versions of
it, their "same" type **stops unifying**: in a typed language a `Service`
from `lib v0.1.6` is a *distinct type* from `Service` from `v0.1.7` (a
mismatch that reads as impossible — "expected `Service`, found
`Service`"); for a C-ABI dylib, two versions in one process is
duplicate / incompatible symbols and undefined behaviour.

- **One version in the graph.** Pin the shared lib **once** — a single
  workspace dependency the whole tree inherits — not per-crate pins that
  drift apart. When a transitive dep forces a second version, that's a
  diamond-dependency conflict to **collapse** (bump everyone to one
  version), not to ship two of.
- **Aligning is part of the rollout.** Publishing a new shared-lib
  version ([§28.1](#roll-out-shared-fix)) isn't done until **every**
  co-built consumer is on it — a half-aligned graph (some crates old,
  some new) won't compile, or links two copies. "Fleet alignment" = one
  version everywhere, in lock-step.
- **Use the version stamp to find the stray.** When a "same" type
  mysteriously won't unify, diff what each consumer actually resolved
  ([§28.3](#lib-version-stamp)) — the odd one out is the second version
  hiding in the graph.

The generalisable rule: **a shared lib that defines boundary-crossing
types resolves to exactly one version across a build's dependency graph —
pin it once, align every consumer on the same version, and treat a second
version as a conflict to collapse (its "identical" types don't unify, two
C-ABI copies are undefined behaviour), never a coexistence.** See
anti-pattern #105.

<a id="immutable-version-tag"></a>
### 28.5 A published version tag is immutable — burn a bad one, never move it

Consumers pin a shared lib by a **version tag** (a cargo/go git dep on a
tag, a `pip install git+…@tag`, an npm version). That tag is a
**contract**: the version *number* maps to exact *content*, forever. Break
that and every consumer either silently gets the wrong bytes or fails to
update — and re-pushing doesn't fix it. Two ways it breaks, both shipped
in one publish cycle:

- **A moved tag.** Re-pointing an existing tag at new content doesn't
  propagate: a consumer's resolver **caches `tag → sha`** (in its lockfile,
  its registry, a CI mirror), so a moved tag is *"unusable"* — the
  consumer keeps the old sha or hits a mismatch, and your fix never lands.
  `git push` itself refuses a moved tag without `--force`; that refusal is
  the system telling you the truth.
- **A poisoned tag.** A **broken / partial publish** tags the wrong commit
  (or the tag push lands but the content push doesn't — the
  [§23.5](delivery-rules.md#reverify-before-retry) "a dropped write isn't a
  failed write" family), so the tag *exists* but carries wrong/stale
  content. Consumers pin it and silently build the wrong code under a
  trusted number.

The fix is the same in both cases: **burn the version number and cut a
fresh, higher one.** Don't re-point or re-push the bad number (`v0.1.12`
poisoned → re-cut as `v0.1.13`, never `v0.1.12` again) — only a *new*
number forces every consumer that already cached the bad one onto clean
content. A burned number is dead forever.

Then **verify the tag from a clean fetch before consumers depend on it** —
a fresh clone / the resolver's own fetch, confirming the tag carries the
intended version stamp ([§28.3](#lib-version-stamp)) and content. A local
*"publish succeeded"* is not *"the remote tag has the right bytes"*
([§23.5](delivery-rules.md#reverify-before-retry)); the publish is a
[finish-the-operation](delivery-rules.md#finish-the-operation) handoff that
isn't done until the tag the consumer reads is correct.

The generalisable rule: **a published version tag is immutable — never
move or overwrite a tag consumers pin (their resolver caches `tag → sha`,
so a moved tag breaks or silently no-ops); if a tag was cut wrong
(poisoned with wrong content, or a partial/broken push), burn that number
and cut a fresh higher one rather than re-pointing it; and verify the
tag's content from a clean fetch before announcing it.** See anti-pattern
#115.

---

<a id="control-flow-dispatch"></a>
## 29. Control flow: prefer dispatch/polymorphism over deep conditionals

> **The standing rule (unomove house rule)** — **try not to use `if/else`. If you find you must, that is the signal to redesign the code, not to add the branch.** A conditional that decides *business behaviour* is a design you haven't found yet; go find it.
>
> **When this fires** — you're writing (or growing) an `if/elif` ladder, a `switch`, or nested conditionals that branch on a *type, kind, mode, provider, or state*.
>
> **Do** — replace the branch with **data**: a dispatch table (`{key: handler}`), a polymorphic strategy, or an explicit state machine. Keep `if` only for a **guard clause** (early-return on a bad input) or a **boundary fallback** (I/O / parse / external call), and centralise those. Everything else is a redesign waiting to happen.

Deep `if/else` is a design smell, not a coding style. Every branch a
function grows is a state a reader must hold and a test must cover; a
ladder that dispatches on *what a thing is* (`kind == "x" … elif kind
== "y"`) hard-codes the set of cases at that one call site, so adding a
case means editing every ladder that enumerated it — and the day two
ladders disagree, the behaviour forks silently. **If you find yourself
needing many branches, that is the signal to redesign, not to add
another `elif`.**

- **Branch on a value → use a table.** `{kind: handler}` (or a registry
  a plugin appends to) turns N branches into one lookup. The set of
  cases becomes *data* the whole system shares — one source of truth —
  so a new case is one entry, not a sweep across every site that
  switched on it.
- **Branch on a type → use polymorphism / a strategy.** When each case
  carries its *own* behaviour (an engine that knows its own
  price/resolve/failover, a tool that knows its own dispatch), give each
  a common interface and let the object answer — the conditional
  dissolves into the type. This is *replace conditional with
  polymorphism*; the payoff is the caller stops knowing the cases.
- **Branch on a lifecycle → make the state machine explicit.** A tangle
  of `if started and not stopped and not draining …` is a state machine
  in hiding; name the states and the legal transitions (§3.6) so the
  illegal combinations can't be expressed.
- **The `if` that stays.** Two uses are legitimate and must *not* be
  polymorphed away: a **guard clause** that early-returns on a bad or
  absent input (it flattens nesting), and a **defensive fallback at a
  boundary** — I/O, a parse, an external API — that must never raise
  (§7, §25). Keep those, but **centralise them in one boundary helper**
  instead of scattering the same `try/except` / `getattr(…, default)`
  across every caller; a hardening branch copied into twenty functions
  is the same duplication §28 warns about, in conditional form.

The test: if a branch decides *business behaviour*, it wants to be data
or a type; if it only *protects* against a bad input at an edge, it's a
guard — keep it, and keep it in one place.

---

<a id="how-to-add"></a>
## Appendix: How to add to this rule book

This file is a small textbook, not a manual. Adding to it has a
shape; following the shape keeps the book usable as it grows.

### When to add a rule

Add a rule when **you've corrected an assistant on the same shape
twice across sessions**. Once is a one-off; twice is the rule
trying to surface. Below that bar, the right place is project-
local notes (`CLAUDE.local.md` in the codebase the bug lives in).

### Where to put it

- **Behavioral guidance** (how to write code, how to think about
  changes) → `CLAUDE.md`. Keep that file short.
- **Domain rule** (naming, units, safety, schemas, supervision,
  UI, messaging, Rust, build, network, …) → a section of
  `engineering-rules.md`. Pick the matching existing section
  first; only add a new top-level section when the rule
  genuinely doesn't fit any of the existing ones — and a new
  top-level section **opens with a `> When this fires … / Do …`
  header** (the gate requires it; see any §).
- **Recurring bug** (named by *shape*, not by service) →
  `anti-patterns.md`.
- **Pre-done check** (something to verify before claiming done)
  → `review-checklist.md`.

### Section anchors, not section numbers

Every H2 has an `<a id="…"></a>` slug right above it; every
non-trivial H3 has one too. **Cross-references in the body use
slug links, never bare `§N.M`**:

```
Bad:  See §8.19 for the rule.
Good: See [§8.19](ui-rules.md#wire-state-mirror) for the rule.
```

The display text can include the section number for human
reference; the link target is the slug. Renumbering doesn't
break refs.

When you add a new H2 / H3 that the rest of the file will
reference, add an `<a id="…"></a>` line **right above** the
heading. Pick a slug that names the *concept*, not the section
number — slugs survive renumbering.

### Shape of a rule

A rule earns its place by being **observable and falsifiable**.
Each entry should answer:

1. **What is the rule?** One sentence imperative form.
2. **Why does it exist?** What bug class does it prevent, or
   what bug have we shipped that it would have caught.
3. **What's the canonical shape?** A short code example showing
   the right and (where useful) the wrong shape.
4. **What's the failure mode if you ignore it?** Named — *"the
   feature is off on every fresh install"*, *"the asyncio loop
   freezes for 600 ms a second"*, *"the vehicle accepts a
   malformed CAN command"*.

If the rule can't answer #4, it's not a rule yet — it's a
preference. Don't add preferences.

### Shape of an anti-pattern

Four lines, in this order, every time:

```
**Symptom.**   What the operator sees.
**Looks like.** What the first suspect would be.
**Actually is.** The root cause.
**Fix.**       Inlined code or one-paragraph procedure.
```

Three lines beats ten. Fast recognition next time is the point.

### Shape of a checklist item

A check is one bullet beginning with `- [ ]`, phrased so the
reviewer can answer **yes / no / not-applicable**. Avoid
"consider whether …" — that's an essay, not a check.

```
Good:  No `widget.close()` from a worker thread.
Bad:   Consider Qt threading implications carefully.
```

If a check belongs to a sub-group (immediate-mode vs retained-
mode UI, e.g.), put it in a sub-group section heading rather
than splattering it inline with conditions.

### Keep it codebase-agnostic

The book describes **shapes**, not specific projects. No repo
names, no file paths a particular codebase uses, no hardware,
no vendor SDKs that aren't already public vocabulary (Linux,
Qt, ImGui, Cap'n Proto are; the operator GUI project's name
is not).

If you find yourself naming a project to explain a rule, restate
the rule on the property: not *"the operator GUI does X"* but
*"any operator-facing UI that …"*.

### When to renumber

Almost never. The TOC at the top maps section numbers to slug
anchors; the cross-refs in the body use slugs. Adding a new
section between existing ones means: pick the next free number
out of order (e.g., insert `§10.5` between `§10` and `§11`
rather than renumbering `§11–§22`), and add it to the TOC at its
logical position.

If a sequence genuinely needs reordering — rare — fix the
numbers in the headers and in the TOC. **Don't fix the body
refs**: they're slugs, not numbers.

### Keep the file under one head's worth of context

Two splits already happened for size: the UI section (§8 →
[`ui-rules.md`](ui-rules.md)) and the book's second half (§11–27 →
[`delivery-rules.md`](delivery-rules.md)). The discipline that drove
them: **keep each rule file near one read-through.** When a file
outgrows that, move a cohesive block of sections to its own file and
leave a one-line **stub** here for each (the §-number + its `When this
fires` header + a pointer), so the section numbering stays contiguous
and every slug anchor still resolves — then register the new file in
[`check.py`](check.py). The TOC keeps the same shape; the split is
transparent to slug links.

### The standing rule

> **Less steering, not more.** Every entry you add has a cost: a
> future reader has more text to skim, more refs to chase, more
> rules to honour. Earn each one.
