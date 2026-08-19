# Self-review checklist

Run this on the diff **before** telling the user "done." It takes
under a minute and catches things that have bitten us in code
review.

If a box doesn't apply (you didn't touch a safety-critical path,
the change is doc-only, etc.), skip it. If you can't honestly tick
a box that does apply, fix the thing or surface it — don't quietly
ship past it.

The checklist assumes you've read [`CLAUDE.md`](CLAUDE.md) and
[`engineering-rules.md`](engineering-rules.md). It is project-neutral
on purpose.

---

## A. Scope (every change)

- [ ] **Every changed line traces to the user's request.** No
      "improvements" to adjacent code, comments, or formatting
      that the user didn't ask for.
- [ ] **No speculative abstraction.** No interface for a single
      caller, no flag for a hypothetical second case, no config
      hook nobody wired.
- [ ] **Critical-path logic is reused, not reimplemented.** If my
      change needs a decoder / transform / safety gate / codec that
      already exists, I wired to the **existing** shared module
      (factoring it out if this is its second caller) — I did not
      write a second copy. Two copies of critical logic drift
      silently. (§28, anti-pattern #69.)
- [ ] **New code defaults to Rust; reusable logic goes in `unolib`.**
      If anything I wrote is new code, it's **Rust unless I can name
      why it can't be** — Python only for UI (Qt/imgui), ML / tensor
      runtime glue, or a true one-off script; a new *service* in
      Python is justified, not assumed. Reusable logic goes in
      `unolib` (checked "is it already there?" first), by published
      version — not a copy. I didn't churn-rewrite *working* Python on
      a hunch. (§9, §28.)
- [ ] **If I hot-edited a deployed box** (a live `.py` swap, an
      in-place server edit): the edit is **captured back to the repo
      same-day** — a hot-edit is a loan, and the next deploy erases
      work that exists nowhere else. Before deploying to a long-lived
      target I **diffed the target against the repo** (drift-ahead is
      a fix to capture, never a silent clobber), and marked the live
      edit (`.bak` alongside). (§13.3, anti-pattern #80.)
- [ ] **No runtime dependency hand-installed into a build-regenerated
      location** (a venv's `site-packages`, `target/`, a cache the build
      wipes) — it's erased by the next build. Deps go through the
      **build's manifest / lockfile** (every rebuild reinstalls them); a
      code change + its new dependency ship as **one paired unit** via
      the repaved path (never a durable half + a wiped half); verify on
      the **rebuilt** form by **loading** the dep, not "the file's
      there." (§13.4, anti-pattern #95.)
- [ ] **Shared content that must merely be present** (coding rules,
      lint/CI config, a `.proto`/schema, templates) is **vendored as
      committed copies** with a path-limited copy-in step — **not a git
      submodule**, which breaks builds on a missing `--recurse-submodules`
      (empty dirs), a `file://`/local-mirror URL (blocked by git 2.38+,
      CVE-2022-39253: `transport 'file' not allowed`), or an unreachable
      URL. Submodules / prefix-deps are reserved for **versioned code**
      with its own lifecycle (§13.2). (§13.5, anti-pattern #122.)
- [ ] **If I fixed shared / `unolib` code, I rolled it out — not just
      the repo I'm in.** The fix is done at *rollout*, not compile:
      published the lib version, fleet-bumped every consumer's pin
      (one command), and can confirm by version stamp which carry it.
      Pinned per consumer (didn't float). (§28.1, §12.1.)
- [ ] **A published version tag I cut is immutable.** I never moved /
      overwrote a tag a consumer pins (resolvers cache `tag → sha`, so a
      moved tag breaks or silently no-ops). A tag cut wrong (poisoned with
      wrong content, or a partial/broken push) got a **fresh higher
      number**, never a re-push of the bad one. After publishing I
      **verified from a clean fetch** that the tag carries the intended
      stamp (§28.3) + content before consumers depend on it. (§28.5,
      anti-pattern #115.)
- [ ] **If I changed a cross-language shared lib** (a C-ABI / `unolib`
      lib consumed from Rust/Python/C++): I edited the **published client
      surface** consumers actually depend on (not just the internal
      `crates/` workspace), classified the change's **ABI impact**
      (additive → no reship; a wire/header/format bump → reship every
      triple + re-pin every consumer), kept the **schema on the consumer
      side** (each owns its own, lineage-guarded), and **verified by
      building a consumer** against the new prefix — not the lib's own
      tests. Config resolves **independent of the CWD** (register the
      table at init, don't read `./config/…`). (§28.2, anti-pattern #101.)
- [ ] **If I rebuilt a shared lib**, every triple was rebuilt + published
      from **one version atomically** (not some arches and not others),
      the artifact **self-reports its version** (an exported
      `…_version()` / embedded string / sidecar, with `git_sha`), and I
      **verified the version actually loaded/linked** (in the prefix and
      what the consumer resolves) — not the source `Cargo.toml`. Partial
      rebuilds across a shared prefix are the default way lib versions get
      mixed. (§28.3, anti-pattern #103.)
- [ ] **A shared lib that defines boundary-crossing types resolves to ONE
      version** in the dependency graph — pinned **once** (a single
      workspace dep), every consumer **aligned** to the same version. A
      second version (a divergent pin or a transitive dep) is a
      diamond-conflict to collapse, not coexist — its "same" types don't
      unify (`expected Service, found Service`) and two C-ABI copies are
      UB. (§28.4, anti-pattern #105.)
- [ ] **If I published a `unolib` version, it ships BOTH linux
      x86_64 and aarch64** — zigbuild the arch I'm not on, lay out per
      `(os, arch)`, and validated **each** by loading it (not by the
      build exiting 0). A single-arch publish strands the other half
      of the fleet. (§28.1, §11.)
- [ ] **Cross-platform builds run off a declared target matrix** —
      each target (`host` / `rig` aarch64-linux / `server` x86_64-linux)
      selected by **one reproducible toolchain-activation command**
      (pinned `(triple, sysroot, linker, features)`), not ad-hoc flags
      per machine; the **whole matrix** is built/published, not just the
      host. And I **cross-built for the real target early** (CI /
      pre-merge) — the host build is not representative, so target-only
      code (cfg branches, ABI / struct layout, libc features) compiles
      clean on the host and breaks only on the target's triple; then I
      **ran on the target** (§13.1). (§11.3, anti-pattern #97.)
- [ ] **A binary that links a shared lib finds it via a baked
      relocatable rpath** (`$ORIGIN/../lib` / `@loader_path/../lib`, or a
      consume-time activation step), **not** a runtime `LD_LIBRARY_PATH` /
      `DYLD_*` env var (an env-var crutch, §7.5) or a forgotten post-hoc
      `patchelf`. Verified by running from a **clean shell** (no lib-path
      env set) on the deployed layout — `ldd` / `otool -L` resolve every
      lib via the rpath, not "not found." (§11.4, anti-pattern #99.)
- [ ] **No dead code I introduced.** Imports, variables, helpers
      that became unused because of *my* changes are removed.
      Pre-existing dead code is *mentioned*, not deleted.
- [ ] **No leftover copies of files I edited** (`.bak`, `.orig`,
      `foo_old.py`, `foo_new.py`).
- [ ] **No new TODO / FIXME / XXX with my name on it.** If the
      work is incomplete, the user gets told, not the source tree.

## B. Naming (every change touching user-visible strings)

User-visible strings live in the uno-namespace.

- [ ] Every user-visible identifier I added or renamed is in the
      uno-namespace.
- [ ] Mode / toggle labels don't collide with adjacent vocabulary
      already in use elsewhere in the build system or runtime
      (e.g. don't reuse "debug" for an unrelated user-mode flag
      when "debug build" and "debug logging" already mean
      something).
- [ ] Standalone components stay standalone — no dependencies
      added to compose them back into other stacks "for less
      divergence."
- [ ] Any non-uno identifier I had to keep for byte-level
      compatibility lives in a low-level constants file with an
      inline comment explaining the constraint.
- [ ] **If this is a ground-up rewrite of a legacy stack**, it's a
      **clean break, not a fork**: the new namespace + conventions run
      all the way through (internal modules, types, layout, CLI, log
      tags — not just user-visible strings), it depends on the legacy
      only at **one deliberate narrow seam** (the shared wire as
      lineage-guarded **data**, §28.2 — never the legacy's code/types/
      build), defaults are re-stated from its own source (§2.3), and the
      clean-break decision is **recorded** (§18.3) so drift back into the
      legacy's names/layout/identity is a flagged regression. (§1 clean
      rewrite, anti-pattern #104.)

## C. Units & schemas (every change touching a message or API field)

- [ ] New / modified fields are SI (m, m/s, rad, s, …) unless the
      name carries an explicit non-SI unit (`*_mph`, `*_deg`,
      `*_kmh`, …).
- [ ] Field names are unambiguous in the message's context.
      `speed` alone is wrong unless the unit is universally
      obvious.
- [ ] A **queue / socket consumer poll loop** (CAN, msgq, socket)
      **drains all available items per wake** (blocking read for the
      first, non-blocking drain for the rest), not one item per
      tick — one-per-tick caps intake at the wake rate and silently
      drops a faster producer. (§5.2, anti-pattern #64.)
- [ ] If I subclassed a **framework interface whose base default
      *does* something** (publishes, sends, actuates): the subclass
      wiring is explicit and verified (the sibling-pattern class
      attribute / registration), and a forgotten wiring fails **loud**
      or falls back **inert** — a base that publishes is a second
      publisher and kills the victim process on a single-publisher
      transport, latent until the other gate opens. (§5.3,
      anti-pattern #81.)
- [ ] No **publisher opened on a topic this process only consumes** —
      on a claim-on-create transport, *constructing* a publisher claims
      the topic and revokes the real producer, so a "keepalive
      publisher" (to make subscribe work) silently assassinates the
      producer in a start-order-dependent way that looks like a flaky
      transport fault. Subscribers attach-or-create; a daemon dying
      `send failed (-1)` right after another starts = a publisher
      collision on one of its output topics. (§5.3, anti-pattern #92.)
- [ ] A **bridge / forwarder / router** derives the set it serves
      **by type / interface** (`isinstance` the subscriber base, scan the
      registry), **not** a hand-maintained parallel name-list — a second
      list silently drifts when a component is added, drops it without
      erroring, and the missing data is misattributed to the source. When
      a consumer shows **no data** for a healthy producer, check the
      bridge forwards it (anti-pattern #1) before blaming the source.
      (§5.4, anti-pattern #96.)
- [ ] Any **timestamp that downstream correlates/syncs** across
      streams is stamped from **one shared monotonic clock, one
      epoch, one unit** (`CLOCK_BOOTTIME`/`*_ns`) at the common
      boundary — not a per-pipeline/per-process clock (e.g.
      `GST_BUFFER_PTS`) that desyncs the streams and stalls the
      consumer. (§2.1, anti-pattern #48.)
- [ ] A **camera calibration / intrinsics / homography** is pinned
      to **one resolution — the consumer's (model's)** — across
      capture, calibration, runtime, and overlay; a stage at a
      different res **scales** the intrinsics rather than passing
      the raw matrix; the resolution is stored in the artifact and
      checked on load. A res mismatch silently skews geometry.
      (§2.2, anti-pattern #66.)
- [ ] If this is a **fork / port onto different hardware**, every
      **inherited hardware constant** (device/sensor table, camera
      intrinsics / FOV / mounting, sensor scaling, calibration
      defaults) was **re-derived from the actual unit**, not trusted
      from the upstream source — an inherited default that loads
      without error is *not* validated, it silently encodes the wrong
      device. (§2.3, anti-pattern #70.)
- [ ] No field was reordered, renumbered, or deleted. Old logs /
      persisted data must replay against the new schema.
- [ ] No type-narrowing of an existing field — added a new field
      with the new type and migrated consumers, instead.
- [ ] If a new topic exceeds the default ring size, an explicit
      size hint is set (per-deployment config, workspace baseline,
      or ad-hoc at construction — in that order of preference).
- [ ] Bulk payloads (camera frames, tensors, multi-MB blobs)
      travel **by reference**: a small descriptor on the bus, the
      bytes in shared host/GPU memory both ends map — not copied
      through the bus per frame. Device buffers shared via VMM
      fd-handoff (pass the fd once); clone a pooled/reused buffer
      before retaining it; build the GPU context after `fork`.
      (§5.1.)

## D. Safety-critical (every change to software with direct actuator authority)

If the code under edit can write a command that reaches a physical
actuator without a hardware backstop in between, treat the change
as safety-critical.

- [ ] Named the actuator path the change can reach and at least
      one failure mode considered.
- [ ] Did not widen actuator / accel / jerk limits, shorten
      safety timeouts, or disable interlocks on the strength of
      simulation alone.
- [ ] Did not refactor a load-bearing safety handshake (engage,
      prepare, state-transition) into a generic state machine
      without verifying on real hardware.
- [ ] Asked the user before merging this specific path, even if
      they previously approved an unrelated change. Approval is
      per-path, not blanket.
- [ ] Secondary control surfaces (web GUI, mobile companion,
      any "remote-of-the-remote") **default to view-only**.
      Control commands are dropped (HTTP 204, no-op) unless the
      client has explicitly opted in via a "TAKE CONTROL" toggle.
- [ ] E-stop / max-brake-latch / kill-switch endpoints work in
      **every** mode, including view-only. They include a
      minimum hold period so a single tap commits even if the
      client disconnects immediately after.
- [ ] Every safety watchdog (input-stale timeout, link-dead
      timeout, gate-stale timeout) has an inline comment naming
      the failure modes it catches and why this number. The
      constant alone tells future-you nothing.
- [ ] A feed used for **remote takeover / teleop** is
      **latency-bounded and latest-wins**: single-slot latest
      frame, receive backlog + decode queue capped to a small time
      budget (drop stale → newest), fps/resolution traded for
      recency, and the tile marked **stale** when the newest frame
      ages out. No smoothing playback buffer on a control feed.
      Quality adapts via a **closed loop on an *unmaskable* signal**
      (drop-rate + receiver RTT — not a buffer a frame-dropper holds
      low), not static presets. Safe levers = **bitrate + fps +
      stream-count**; **encode resolution is fixed at startup, never
      re-capped live** (it re-inits the HW codec → wedges a shared
      engine under load, #65). The path (§10.1) decides **how many**
      streams; the **send buffer is hard-capped** (drop even
      keyframes) so a slow link degrades fps, not latency. (§3.4,
      anti-patterns #52, #65.)
- [ ] An **adaptive controller over a shared, capped resource** (ABR on
      one uplink, clients on one pool/limiter) closes the loop on the
      **total budget** and sets each consumer's share = budget ÷ active
      consumers — **not one independent per-consumer loop** (they each
      probe into the shared headroom, overshoot together, and flap on a
      ~minute cycle). Concurrent sessions divide the *same* budget;
      dead-band + asymmetric smoothing at the shed/restore boundary; one
      writer owns the throttle (§5.3). (§3.4, anti-pattern #94.)
- [ ] A **link that carries live commands to an actuator** (teleop
      joystick, remote-control channel, setpoint stream) **safe-stops on
      staleness** — every tick it checks command freshness on an
      **unmaskable** signal (the command's timestamp / a sender
      heartbeat, **not** a `connected` flag) and, past a small staleness
      bound, bails to a **validated safe state** (idle / coast-stop /
      neutral / brake); it does **not** keep replaying the last command
      after the link drops (a held command is a runaway). Centralized in
      a deterministic watchdog where possible. (§3.7, anti-pattern #98.)
- [ ] A **feedback controller closes on the variable it controls**, not a
      convenient **proxy** that can decouple (motor rpm as a stand-in for
      ground speed, an encoder under backlash) — a decoupling proxy makes
      the loop **hunt / limit-cycle** (a ~constant-frequency actuator
      pulse with the plant not responding). Sense the real variable, or
      cross-check the proxy against an independent signal and **fail safe**
      (§3.5) when they diverge. (§3.8, anti-pattern #100.)
- [ ] An **adaptation / recovery lever that re-initializes shared
      hardware** (a live encode-res re-cap, a device re-open) is
      kept out of the live loop and **validated under realistic
      concurrent load, not at idle** — watching the *shared*
      resource, not just the lever's own output. (§9.9, anti-pattern
      #65.)
- [ ] A **calibration / config / map feeding a safety-critical
      consumer** is validated before it reaches it: the sanity gate
      rejects on **co-occurring** symptoms (not one threshold —
      corruption stacks, real data is consistently skewed), **fails
      safe to a known-good default loudly** on reject, and the
      deploy **verifies on the target** that the value was accepted
      at live conditions ("deployed" ≠ "applied"). (§3.5,
      anti-pattern #58.)
- [ ] A **periodically-recomputed decision** (a plan, route, selection,
      mode, level) **re-validates the current one and holds it** while
      still good — it does **not** recompute from scratch every tick
      (near-equal solutions flap, and consumers chase a jittering
      reference). It recomputes only on a **real trigger** (became
      invalid / a genuinely new input / a *sustained*, not transient,
      failure) and switches only past a **margin / dwell-time**
      (hysteresis). (§3.6, anti-pattern #93.)

## D.bis. Money / privileged actions (multi-user app; the server is the trust boundary)

- [ ] No client-sent value with **money or authorization** meaning
      is trusted — the server **recomputes** it from authoritative
      data (the record's amount, the user's role) and ignores the
      posted one, keyed on a **stable identity** (opaque id, not a
      display name). The UI is not a security boundary. (§26.1,
      anti-pattern #60.)
- [ ] Every privileged action is **authorized server-side**, not by
      hiding the button; rendered user content is escaped (XSS);
      money/message-sending endpoints are rate-limited. (§26.1.)
- [ ] **Untrusted data crossing into an interpreter** (HTML/JS, SQL,
      a shell, a template) is **encoded for that sink** — output
      auto-escaped for the exact context (HTML body vs attribute vs JS
      vs URL); input **parameterized** (bound query params, an **argv
      array** — never `shell=True` or string-built SQL), not
      interpolated raw. Encoding is on the **server**; **stored** values
      are treated as untrusted when rendered (stored XSS). (§26.6,
      anti-pattern #118.)
- [ ] A **money-moving or irreversible** action (payout, bulk
      delete, role grant, purge) has a **second approver** (requester
      ≠ approver), an enforced **cap** on blast radius, and a
      tamper-evident **audit trail**. (§26.2.)
- [ ] An **access grant** (operator onboarding, an SSH grant, a
      service account) authorizes the requester's **own public
      key** on a **least-privilege account** (forwarding-only /
      read-only) behind an approval gate — **never** a shared
      password or copied private key. A shared secret can't be
      revoked per-person, loses attribution, and forces a
      fleet-wide rotation on any leak/departure. (§26.4,
      anti-pattern #85.)
- [ ] **At fleet scale**, access isn't a key on every host's
      `authorized_keys` (O(identities × hosts) — it drifts and
      revokes per-host). Hosts **trust a central SSH user CA**
      (`TrustedUserCAKeys`) that issues **short-lived per-identity
      certs**: enroll once → reach every host, certs expire on their
      own, a **KRL** revokes one identity fleet-wide. The CA key is
      guarded as a fleet-wide secret. (§26.5, anti-pattern #114.)

## E. Process supervision (changes to a long-running process)

- [ ] New process has a gate predicate (feature flag, role,
      device-file existence, peer health, …) — not defaulted to
      "always run."
- [ ] Inline rationale comment if the predicate isn't obvious.
- [ ] Transient-resource opens (TTY, socket, GPU context) inside
      a daemon use **retry with loud WARNING**, not catch-and-
      sleep-silent. A daemon that sleeps on error and never
      reports it is the worst kind of "live" — the supervisor
      thinks it's healthy; the topic publishes nothing.
- [ ] For wrapped exceptions (`SerialException` wrapping
      `PermissionError`, etc.): the `except` clause catches the
      wrapper class or matches the inner error message —
      not the bare inner exception.
- [ ] New/removed service goes through the **one declarative
      process table** (not a second launch script / unit / hand
      start); the supervisor owns its lifecycle. A removal also
      greps out its consumers, config keys, bootstrap steps.
      (§6.1, anti-pattern #31.)
- [ ] There's a way to diff **running vs. declared** services
      (a reconciliation probe) — orphans and silent-dead daemons
      are findable by command, not memory. (§6.1.)
- [ ] Restart policy matches how the process exits: **`Restart=always`**
      when a clean (0) exit can be a boot-race / exit-to-restart
      pattern (`on-failure` skips exit 0 → service stays dead after
      a reboot). If the unit is generated by a bootstrap script,
      the **template** was edited, not the live `/etc` copy. (§6.2,
      anti-pattern #45.)
- [ ] A fault that only a **power-cycle** clears (wedged USB / hung
      sensor firmware / fence-blocked GPU) is **escalated loudly**
      ("reboot required") after bounded retries — not
      software-restart-looped in silence. (§6.2.)
- [ ] A **must-always-run** service isn't silently defeated by
      systemd's **start-limit**: a fast-flapping unit (default 5
      starts/10s) wedges in `failed` *despite* `Restart=always`. Set
      **`StartLimitIntervalSec=0`** in the **`[Unit]`** section
      (no-op'd in `[Service]`), and keep **`TimeoutStopSec` generous**
      so a slow stop doesn't escalate to `failed(timeout)` and burn the
      restart budget. First check for "stale/frozen data" from a
      supervised service is `systemctl is-active <unit>` (stale
      `/dev/shm` + a rate probe read the dead producer's last values).
      (§6.2, anti-pattern #120.)
- [ ] A subscriber/client that **reconnects or resubscribes on
      silence** closes the old handle (reassigning a variable does
      **not** free a shared IPC slot / fd / pool connection) and
      **caps retries per silence episode**, then defers to restart —
      an unbounded loop leaks a *capped* shared resource and can
      freeze every consumer, not just itself. (§6.4, anti-pattern
      #62.)
- [ ] A service that uses **`/dev/shm` / POSIX IPC** and runs as a
      user isn't tied to an interactive login — it's a **system
      service** or the user has **`enable-linger`**, so logind's
      default `RemoveIPC=yes` doesn't reap its IPC at last logout
      (producer keeps the deleted inode, new subscribers get a fresh
      empty one). (§6.5, anti-pattern #63.)
- [ ] A daemon that **opens a device** (serial/USB TTY, CAN, i2c/GPIO,
      GPU) has its OS access **granted in the version-controlled unit**
      (`SupplementaryGroups=dialout` / a capability / a udev ACL) — the
      service runs with the **unit's** groups, not my interactive shell's
      ("I'm in `dialout`" ≠ the service is). A device-open
      `PermissionError` surfaces **loudly** (retry + WARNING / operator
      indicator), never a silent sleep or a one-shot crash that strands
      the daemon — else it reads as "sensor invalid / no data" and the
      hardware gets blamed. (§6.10, anti-pattern #119.)
- [ ] If a **manager/supervisor imports every registered process /
      plugin module at startup**, a renamed/refactored module name was
      swept across **every** registered module — not just the running
      ones. A module is imported even when its run-gate is off, so one
      stale import is fatal stack-wide. (Diagnose a stack-wide
      crash-loop at the manager's first traceback, not the restart
      spam.) (§6.6, anti-pattern #71.)
- [ ] No client/tool/GUI **`stop`s an always-on service from a
      teardown path** (cancel, app-exit hook, superseding relaunch)
      — an explicit stop defeats `Restart=always`. Teardown
      **detaches**; operate via restart/reload; reserve `stop` for a
      deliberate operator action. And a long-lived client
      **reconnects/re-spawns across the service's restarts**
      (rate-limited), not stale after one connect. (§6.3,
      anti-pattern #51.)
- [ ] A job that must be **always-on / boot-started / self-healing**
      is a **resident service** — a loop (`do-work → sleep → repeat`)
      under **start-on-load** (`RunAtLoad` / `enable`) +
      **restart-on-exit** (`KeepAlive` / `Restart=always`) — **not** a
      cron / calendar / timer one-shot, which has no live process
      between fires, so it can't boot-start or self-heal and reads as
      idle/dead. (A missed-run-is-harmless job stays a one-shot; then
      "idle" is the healthy state.) Verify the live state, not the
      config: it shows **RUNNING with a PID** and respawns after a
      kill. (§6.7, anti-pattern #86.)
- [ ] A process that must run **once per host/resource** (a publisher, a
      writer of a shared slot/throttle, a device-owning daemon) **guards
      single-instance** at startup (a `flock`'d pidfile / `O_EXCL`
      lockfile / bound socket / the topic claim, §5.3) — a second start
      no-ops or **takes-over-and-reaps**, never silently coexists — and
      **reaps the predecessor's orphans**; the read side **expires stale
      pushed values** (§7.6) so a leaked duplicate can't drive forever. A
      stale/flapping shared value = count the writers (`pgrep`/`ss`)
      before blaming the consumer. (§6.8, anti-pattern #102.)
- [ ] The singleton claim is **retried over a bounded window** on
      startup — a just-reaped predecessor's release (a `TIME_WAIT`
      port, a `flock`/pidfile, a device handle) lags a fast restart, so
      a **one-shot** claim hits `EADDRINUSE`/lock-held and exits with a
      false "already running," leaving the service **down**. A live peer
      never releases → still fail after the window. (§6.8, anti-pattern
      #112.)
- [ ] A **teardown / stop path** for a worker (thread, subprocess,
      decoder, async task) stops it **cooperatively first** — a stop
      flag / closed input + a **bounded wait** for it to exit on its own
      (releasing its codec / handle / lock) — and escalates to
      `SIGTERM`/`SIGKILL` **only on a hang**. No immediate forced kill
      mid-operation (it segfaults / corrupts because the worker holds a
      native resource — the teardown path becomes the bug). (§6.9,
      anti-pattern #116.)
- [ ] If I deployed a **rewrite / second daemon** onto a host that still
      runs the legacy (or another implementation) of the same role: the
      singleton guard keys on the shared **resource** (device / topic /
      actuation authority), not the binary name — so a *foreign*
      implementation trips it — and the **onboard step claims the role by
      *disabling* (not just stopping) + reaping the others**: "stopped" ≠
      "gone" (a still-*enabled* competitor re-grabs on next boot), so
      disable in **both scopes** (system + per-user), reap orphans **by
      exe-path, not just the unit**, disable the **whole set derived from
      the stack registry** (not a drifting hand-kept list, §5.4), at **two
      enforcement points** (privileged cutover + per-activation runtime
      claim). Disjoint resources may coexist; sharing one collides —
      symptom on the *victim* (no device / 0 Hz), safety hazard if both
      actuate. (§6.8, anti-pattern #108.)

## E.bis. Silent-default traps (every read of a config / schema field)

- [ ] For any "default ON" config bool: used the explicit-opt-out
      pattern. (Naive `if v is None: return default` **never
      fires** against a KV store that returns falsy on missing
      keys.)
- [ ] No `getattr(obj, "field", 0)` (or `…, False`, `…, ""`)
      against a schema you control. If the field moved or was
      renamed, the fallback returns silently-wrong-but-falsy and
      the caller has no way to tell. Either read the attribute
      directly (let `AttributeError` fire), or use a non-falsy
      sentinel:

      ```python
      _MISSING = object()
      raw = getattr(obj, "field", _MISSING)
      if raw is _MISSING:
          raise SchemaMismatch("field moved — check obj.idx.field?")
      ```
- [ ] No function default that silently substitutes the wrong
      value when the caller forgot to pass (`rpy=(0,0,0)`, identity
      transforms for calibration, zero-extrinsics for "use the
      rig"). Either require the argument or fail loudly on the
      sentinel.
- [ ] **Zero new environment variables that change behaviour /
      config / a code path / a device.** Grep the diff —
      `git diff | grep -nE 'os\.environ|getenv|std::env::var|ENV\['`
      — and for **every** hit confirm it's either a *removed-before-
      merge* debug nudge or a pure location/secret; anything that
      selects behaviour is **banned**, even one boolean, even
      "temporarily." Derive the choice **in code from hardware/role**
      (the `device.py` pattern) or a **checked-in config file**. If
      you believe you need a behavioural env var, you have the wrong
      design — **ask the operator**, don't add it. (§7.5,
      anti-pattern #30.)
- [ ] A consumer that **polls a pushed control / override value** from
      a side channel (a `/tmp` file, a Param, a shared slot written by
      a session / client / adaptive controller) reads its **freshness**
      (mtime / heartbeat / TTL), not just the value — and **expires** a
      stale value (untouched > N s) to a **safe default**, so a writer
      that disconnects / crashes / is reaped without cleanup can't pin
      the consumer to a dead setting forever. (§7.6, anti-pattern #73.)
- [ ] No hardcoded platform-specific storage path in a script or
      bootstrap (`/data/params`, `/etc/app`, a well-known
      absolute path). Write through the same API/resolver the
      reader uses, or run the writer as the same user+env the
      reader runs under. A write to the wrong root "succeeds"
      into an orphan dir the reader never reads. (Anti-pattern
      #25.)

## F. UI (changes to interactive surfaces)

### F.1 Common to both UI models

- [ ] No widget / window-class import below the UI layer boundary
      — services don't know about widgets.
- [ ] Per-stream isolation preserved: independent media streams
      stay in independent processes.
- [ ] Input listeners that drive actuators (WASD, gamepad axes)
      check `io.want_capture_keyboard` (or framework equivalent)
      **before** consuming the key. A stray "w" in a search box
      must not reach throttle. (Safety rule — see §D.)
- [ ] Streaming-output producers coalesce publishes (≤ N events
      OR ≤ T ms; typically 10 / 50 ms). No per-line snapshot
      copies.
- [ ] Views backed by unbounded buffers (logs, tables) virtualize
      to the visible viewport.
- [ ] GPU texture uploads gated on a sequence-number / dirty
      flag, not unconditionally every frame.
- [ ] Auto-scroll for streaming views captures scroll position
      **before** drawing this frame's content (otherwise auto-
      scroll fights an operator who scrolled up).
- [ ] Persistent config writes use tmp + rename (atomic), not
      direct `open(...).write(...)`.
- [ ] Control widgets render from the **publisher's snapshot of
      what's on the wire**, not from local input. (See
      anti-pattern #21.) The widget exposes a "frames sent"
      counter so operators can localise "GUI not sending" vs.
      "rig not receiving."
- [ ] A **control action** (engage / arm / set-mode / push a
      setting) confirms by **reading back** the authority's actual
      state — not by the write returning success. An optimistic latch
      lies the instant the target rejects same-tick (a blocking event,
      a re-latch); reconcile-poll catches a *later* clear, short period
      for a safety-relevant state. A safety-state indicator never
      asserts a state the system isn't in. (§8.27, anti-pattern #75.)
- [ ] An **operator action that drives a multi-step flow** (onboard,
      link, enroll, deploy, retry) is **idempotent and re-runnable** —
      a second press **converges** (skips what's done via a read-back,
      retries what failed), it doesn't error ("already exists") or
      double-apply. **Re-run is the one-click recovery path** (always
      available, labelled for what it does), it shows the **verified
      end-state** and which step is outstanding, and auto-attempts on
      open where that's safe — never a one-shot that strands a half-done
      state the operator can fix only from a CLI. (§8.29, anti-pattern
      #89.)
- [ ] If this is a **telemetry / debug / introspection visualization**
      of multimodal or time-series data (sensors, perception, 3D, point
      clouds, plots, tensors), it **logs to Rerun** (`rerun-io/rerun`) —
      `rr.log()` on the shared clock (§2.1), one viewer scrubs/replays
      every modality, `.rrd` recordings for offline diagnostics (§15) —
      rather than a hand-rolled one-off viewer. (Operator/control
      surfaces stay in the house GUI — Rerun is the introspection layer,
      not the control layer.) (§8.30, anti-pattern #90.)
- [ ] Status banners follow the triad shape: muted on first
      frame (no flash); green/amber/red on the three known
      states; hover-for-detail tooltip carrying the diagnostic
      multi-line; hidden when the steady state is "healthy."
- [ ] Background probes the banner triggers are rate-limited at
      the module level — a per-frame ImGui call must not fire
      HTTP / SSH at the frame rate.
- [ ] Overlays on live media (model polylines, pose markers,
      breadcrumbs) render an explicit "no data" indicator when
      the source snapshot is older than the staleness threshold
      OR when no snapshot has arrived yet. Empty ≠ silent.
- [ ] Workloads that cross ≥ 3 hops (input → publisher →
      transport → bridge → remote) expose per-hop status in the
      live UI so the operator can localise the fault in 5
      seconds. See §8.21.
- [ ] Always-visible surfaces (banners, status rows, lists) show
      masked / friendly forms — hostname, role label, masked IP,
      `•••• 4321` — not raw edge/target IPs, ports, `user@host`,
      **or sensitive user data (PII, bank/card/account, keys)**.
      Sensitive data is masked **by default**; the full value is
      revealed only on an explicit action, and never logged in
      cleartext. "If this is in a customer screen-share, what did
      we just hand them?" (See anti-pattern #24, §8.22.)
- [ ] A field a role isn't allowed to see is withheld
      **server-side** (the API doesn't return it / returns it
      masked), not merely hidden in the UI — UI hiding is one
      DevTools tab from exposure. Display masking and server-side
      authorization are both shipped. (§8.22.)
- [ ] Single-element auto-recovery (one stalled stream of N) is
      **capped** (max bounces / rate limit) and recovers the
      element, not the whole transport. After the cap, leave it
      degraded + surface staleness in the UI. (§10.2.)
- [ ] Liveness for N parallel elements is tracked **per element**,
      not as an aggregate (total frames / queue draining) — a sum
      stays green while one element is dead. (§10.2, anti-pattern
      #33.)
- [ ] Choosing among redundant sources / paths is gated on
      **sustained rate**, not mere "alive": a preferred source
      that's present but scheduler-starved/throttled below its rate
      floor self-demotes to the live fallback (and self-promotes on
      recovery). Aliveness ≠ liveness. (§10.1, §8.20.)
- [ ] UI colours come from the **active theme/style**, not
      hardcoded literals; semantic colours (OK/WARN/ERR) from one
      palette switched with the theme. Verified legible on every
      theme the build ships, not just the one open. (§8.25.)
- [ ] A status indicator's **off / false / absent** colour is chosen
      **per call site** (`off=IDLE|WARN|BAD`) — not a fixed default —
      because the same `False` is *normal* at one indicator (gray) and
      a *fault* at another (a link / authority that should be live →
      red). A fixed off-colour paints a real fault benign (operator
      misses it) or a normal absence alarming (cries wolf). State is
      carried in **shape too** (filled vs hollow), not colour alone.
      (§8.25, anti-pattern #121.)
- [ ] Any recovery / failover / retry mechanism's trigger is
      proven to fire on the live path — not dead code waiting on
      an event the transport no longer emits. (See anti-pattern
      #23.)
- [ ] On a device/kiosk/in-vehicle UI: font size and control
      geometry changed together (paired knobs), not one without
      the other. Controls stay at device scale, not shrunk to
      desktop density. (§8.23.)
- [ ] A size/metric change that invalidates a persisted layout
      bumps the layout's cache key (the `NNpt` in the layout
      name), so existing users recompute instead of restoring a
      now-wrong cached arrangement. (§8.23.)
- [ ] **Layout overlap was fixed in the programmatic layout code
      (dock splits / reserved chrome height), not by dragging
      panels in a running build.** Verified against a WIPED cache
      (`rm` the persisted layout file, relaunch) — not a warm
      session. The test: "would a brand-new machine with no cache
      file see this fix?" If not, you fixed the cache, not the
      code → it comes back. (§8.23, anti-pattern #26.)
- [ ] When rendering a surface the UI framework also provides by
      default (menu bar, status bar, title), the framework's
      built-in is explicitly disabled — owning a surface means
      disabling its default, not drawing over it. (§8.24.)

### F.2 Immediate-mode UIs (snapshot pattern — new house style)

- [ ] Every service that publishes UI state is a
      `SnapshotSource[FrozenSnapshot subclass]`. Producers build a
      complete snapshot then swap; consumers never observe a torn
      write.
- [ ] Every subscriber-style snapshot has a `live: bool` field
      and a liveness timeout. UI shows "stale" when `live` is
      false.
- [ ] Per-frame snapshot cache (`frame_snapshot(svc)`) used in
      views that may read the same service multiple times in one
      frame.
- [ ] Top-level draw functions wrapped in `@safe_view`-style
      containment that catches exceptions, rate-limits the log,
      and **unwinds the begin/end stack to the view's start depth**
      — preferably via the framework's own error-recovery
      (snapshot before, `error_recovery_try_to_recover_state` on
      throw), with its recover-time assert disabled
      (`config_error_recovery_enable_assert=false`), so a scope the
      crashed view left open doesn't abort the frame. (§8.4,
      anti-pattern #13.)
- [ ] Where no framework recovery exists, begin/end UI calls that
      may throw mid-pair use `try` / `finally` to close the
      framework's internal stack at the call site.
- [ ] Workers / threads / subprocesses register a single `stop`
      hook with the lifecycle registry once at construction —
      no per-dialog teardown code.
- [ ] Target / device swap uses the build-new-first sequence
      (new transport up before old torn down).
- [ ] Orphan sweep runs at startup, **before** new workers spawn.
      Match rules conservative (project-root substring, not bare
      command name).
- [ ] Every non-ASCII glyph the UI draws goes through a single
      `SYM` class / module-level constant. No literal `"●"` /
      `"°"` / `"↑"` inside view functions.
- [ ] Every codepoint in `SYM` is verified present in the bundled
      font in the *actual weight* (regular / bold / large) the UI
      renders in. A glyph that's tofu in production fails review.
- [ ] User-visible English strings wrapped in `tr("…")` **as
      written**, not in a retrofit pass. English is the msgid;
      catalogs for other locales only carry translations.
- [ ] No sentence built by concatenating `tr()` fragments —
      translate the whole sentence with a `{placeholder}` so the
      catalog controls word order. (§8.15.)
- [ ] Layout absorbs text expansion: labels/buttons sized to
      content or with slack, not hard-coded to fit only the
      English string (other languages run longer / taller).
      (§8.15.)
- [ ] Adding a language did **both**: the catalog AND a
      glyph-coverage check of the target script against the
      bundled font. No ASCII-fallback/transliteration shim — make
      the font cover it. (§8.14–8.15, anti-pattern #35.)
- [ ] A bundled font is a **static instance** at the intended
      weight, not a variable font (the `wght` axis defaults to
      Thin and a no-FreeType rasteriser can't apply it → gray
      text). (§8.14, anti-pattern #35.)
- [ ] Glyphs emitted by a process that can't import the shared
      `SYM`/i18n module (probe scripts, remote tools) use the same
      literal codepoint the bundled font carries, and any parser
      matches that exact codepoint. (§8.15.)
- [ ] No animation code that scales by a *fixed per-frame step*.
      Use the dt-correct exponential form
      `alpha = 1 - exp(-dt/τ)`, not `alpha = dt/τ`.
- [ ] Animation state objects (`Smoother`, `ColorSmoother`) are
      held per-widget-instance — either at the call site or in a
      module-level cache keyed on a stable widget id.

### F.3 Retained-mode UIs (Qt — legacy maintenance)

- [ ] No `widget.close()` from a worker thread (macOS deadlock).
      Queued `QMetaObject.invokeMethod` instead.
- [ ] No `processEvents()` spin-wait. Async progress dialogs use
      `prog.exec()` + queued-connection updates + a
      `QTimer.singleShot` hard deadline.
- [ ] Nothing **blocks the async event loop** — neither CPU work
      nor a slow **synchronous call** (a sync recv, a DNS/topic
      resolve, a subprocess wait, `time.sleep`, >~100 ms I/O). CPU
      work → `loop.run_in_executor`; a value needed synchronously on
      a hot path → a **background-refreshed cache** the loop reads
      (never an inline probe). Co-resident subscribers *and*
      timer-driven connections (RTC keepalives, heartbeats) must
      keep pace. (Anti-pattern #5.)
- [ ] All workers in the same project share one cancellation
      primitive (a `threading.Event`, a single `stop()` method).
      No `_stop` / `_cancel` / `_abort` drift across siblings.
- [ ] No string introspection of signal names in `closeEvent` —
      if you're tempted, fix the worker class hierarchy instead.

## F.bis. Logging discipline

- [ ] No unconditional `log.*` / `print(...)` inside a tick loop
      or per-event callback. Use **transition + heartbeat**
      (emit on state change, plus every N seconds at idle).
- [ ] Warning-level logs from a degraded code path are
      **rate-limited per source** (once per N s per source key),
      not per-line.
- [ ] No log line inside an asyncio context that fires per-tick
      on inactive subjects — that path starves the writer and
      degrades active subjects. (See anti-pattern #19.)
- [ ] Log levels match operator action: `WARNING` = recovering;
      `ERROR` = operator must act. Don't `ERROR` for a condition
      the code recovers from silently.
- [ ] Each major service publishes `live: bool` + last-update
      timestamp as a snapshot field. UI checks the snapshot, not
      the log, for liveness.
- [ ] A status/health/liveness indicator derives up/down from the
      **primary data link** the producer already streams on (the
      producer publishes its own state as telemetry there) — not a
      **separate out-of-band probe** (a fresh SSH/HTTP handshake per
      poll) that sits idle, gets dropped/throttled by the edge, and
      false-reads "down" while the data flows. A probe may enrich
      detail, never override a live link; hold the verdict with
      hysteresis. (§15.10, anti-pattern #107.)
- [ ] Any always-on signal I touched (CI check, badge, monitor,
      alert) is green-in-the-good-state or removed — not left
      perpetually red "to fix later." A new check lands green.
      (§16.6, anti-pattern #34.)
- [ ] On an embedded target, no high-rate message rides a **slow
      sink** (kernel serial console / `printk`): `console_loglevel`
      keeps `WARNING` spam off the UART (messages still in
      `dmesg`/journald), and a chatty source is rate-limited — a
      slow sink can pin a CPU core and reboot the SoC. A fault that
      names a subsystem is attributed from captured evidence
      (pstore) before reverting the loudest suspect. (§16.7,
      anti-pattern #49.)
- [ ] Each log line is prefixed by **one** owner. If the logging
      framework's formatter already emits the target (the
      component/module name), messages don't *also* hand-prepend a
      `[component]` tag — pick one. One shared init / format across
      the binaries (no bare `init()` next to a custom `Builder`); a
      library installs no formatter; a supervisor forwards a child's
      output verbatim, never re-stamping an already-formatted line.
      (§16.8, anti-pattern #106.)
- [ ] A reader of an **append-only record stream** (a recorded log,
      WAL, event log, framed socket, chunked upload) survives a **torn
      trailing record** from a writer killed mid-append: a mid-record
      EOF at the *end* of the stream is a clean stop (`Ok(None)`) that
      recovers every complete record — not a `read_exact` error that
      discards the whole log. Completeness is checkable (length prefix
      / per-record checksum), and there's a test against a truncated
      stream. (§16.10, anti-pattern #111.)

## G. Embedded sync / deploy

- [ ] If the host architecture differs from the target's,
      rsync explicitly excludes every native-arch binary path.
      A wrong-arch binary clobbers silently.
- [ ] Deploy step is restartable: no orphaned half-state if the
      restart fails midway.
- [ ] Native shared libs (`.so`/`.dylib`/`.dll`/`.pyd`) are
      selected **per platform+arch** at package time (macOS→
      `.dylib`, Linux→`.so`, arm64→arm64), not globbed from one
      shared `lib/`. Verified the linked arch in build/selftest
      (`file` / `lipo -archs` / `otool -L` / `readelf`). No built
      lib committed into source as "the" lib. (§11.1, anti-pattern
      #36.)
- [ ] If the build is **frozen/bundled** (PyInstaller etc.): every
      native lib the bundled extensions `dlopen` is installed in the
      build env (present at collect **and** run time), and I verified
      the frozen artifact by **importing its entry points on a clean
      machine** — not by the build exit code. A freezer silently
      drops a module it can't import at analysis time. (For a
      vendor-pluggable runtime the code doesn't really use, the
      loader/stub that ships the `.so.1` soname is enough — no full
      driver needed.) (§11.2, anti-pattern #50.)
- [ ] Generated build output (`build/`, `dist/`, `target/`,
      `*.o`, wheels, `__pycache__`) is gitignored — not committed
      into the source tree. (§13.)
- [ ] No runtime code assumes the **dev source layout**: a
      deployed module is invoked by **import + entry call**, not
      `python -m` (a compiled/Cython `.so` has no code object);
      `__file__`-relative / sibling-`scripts/` paths try **both**
      repo-root and installed/bundled layouts. Deploy/launch paths
      verified on the **production-built** form (compiled rig /
      frozen bundle), not just a source checkout. (§13.1,
      anti-pattern #53.)
- [ ] No consumer imports from / builds against another project's
      live source checkout or its `build/` dir. Cross-project
      deps go through a **published, version-pinned** artifact;
      bumping the version is a deliberate step. (§13, anti-pattern
      #27.)
- [ ] Anything deployed to more than one target carries a build
      stamp (`git_sha` + `git_dirty`) it reports at runtime
      (`--version` / startup log / status field). On "works on A,
      not B," I diffed the rigs' stamps **before** reading code.
- [ ] A **clean / release** build refuses (or loudly requires an
      explicit override for) a **`-dirty`** source stamp — a dirty
      artifact embeds uncommitted edits and is unreproducible. Dev
      builds may be dirty; anything deployed/published traces to a
      commit. The dirty bit is computed with **`git status
      --porcelain`** (staged + untracked), not `git diff` (unstaged
      tracked only). (§12.1.)
- [ ] If a deploy/onboard **skips the rebuild when "already
      current"**: it gates on **`sha + dirty`** (never the bare commit
      sha), dirty via `git status --porcelain`, so a **staged /
      untracked** edit can't take the committed identity and silently
      not-ship; a `-dirty` stamp never matches (always rebuild); and
      "already current" is confirmed against the version the **target
      reports** (§28.3), not a local assumption. (§12.3, anti-pattern
      #110.)
- [ ] A **broken / flaky build** was fixed by **method, not
      flag-poking**: reproduced from a clean committed tree, inputs
      pinned (lock, toolchain, ref — no env reads), the failing
      **layer** localised (compile → link → package/freeze →
      deploy), built for the target off a live box, and **verified
      on the deployed form** — not just "it went green once." (§27,
      anti-pattern #67.)
- [ ] A **web app served behind a reverse proxy** doesn't hardcode
      the origin root: assets/links/cookies are **relative** or off
      one **configured base path**, external URLs come from
      **forwarded headers** (not the backend socket), and I tested
      it **through the edge** under its real path prefix — not just
      at `localhost:PORT`. (§23.2, anti-pattern #56.)
      `git_dirty: true` on a deployed rig = deployed from an
      uncommitted tree → fix that first. (§12.1, anti-pattern
      #29.)
- [ ] Cross-component compatibility (GUI↔service, rig↔daemon) is
      decided by a **required feature set** (`required ⊆
      advertised`) in **one verdict function** — never by
      comparing a peer's version string (`==`/`!=`/`<`), and no
      hand-mirrored `EXPECTED_PEER_VERSION` constant. The version
      is cosmetic; verify the `service` field too. (§12.2,
      anti-pattern #41.)

## G.bis. Rust / FFI changes

- [ ] Every Rust → Python (or Rust → C) entry point that takes a
      buffer validates **C-contiguous** at the boundary and
      returns a typed error with the fix suggestion in the
      message (`pass np.ascontiguousarray(...)`). UB-silent
      strided buffer is the worst kind of regression.
- [ ] Shared-memory writer-vs-reader access paths have
      distinguishable names (`*_mut` for writer, `*_ptr` for
      reader). A code-review skim must spot a writer call in a
      reader codepath.
- [ ] Any new Rust binding is justified by one of the two [§9 Choosing Rust](engineering-rules.md#rust-vs-python)
      axes: either a **measured** GIL-blocking hot path
      ([§9.1 Hot path](engineering-rules.md#hot-path)) **or** a **named** critical function whose failure
      mode is silent corruption rather than a noisy crash
      ([§9.2 Critical path](engineering-rules.md#critical-path): SPSC ring header, FFI buffer slicing, refcounted
      shared-resource handle, lock-free CAS, actuator-bound
      serialiser, schema enforcement at a trust boundary).
      RSS bloat / "feels safer" / "would have caught it" are
      neither.

## G.quater. Accelerators (GPU / NPU / DLA)

- [ ] Not adding a **second** accelerator context to a process
      that already has one (two CUDA/CL users per frame serialise
      and stall). Isolate the stages or keep the contended one on
      CPU. (§9.5.)
- [ ] Codec / fixed-function work uses the **hardware path**
      (NVENC/NVDEC, V4L2 M2M, VideoToolbox), not a software
      encoder/decoder, on an embedded target. (§9.5.)
- [ ] A model / accelerator stage exposes **output liveness** (a
      heartbeat on its output topic) — a hang is "alive but
      blocked" and a PID check won't catch it; "models loaded" is
      not a liveness signal. Diagnose at `/proc/<pid>/wchan`
      (`dma_fence_default_wait`). (§9.5, anti-patterns #38/#40.)
- [ ] Compute device selected **explicitly** (`CL_DEVICE_TYPE_GPU`
      / named CUDA device), never `DEFAULT`/auto — which can
      silently bind the CPU. The bound device is **logged by
      name** at startup, and the active path announced
      ("CUDA active" / "CPU fallback"). (§9.5 rule 6, anti-pattern
      #40.)
- [ ] Accelerator memory is bounded (model/batch/texture cache)
      to keep headroom — near the ceiling the driver wedges
      instead of returning a handleable error. (§9.5, §8.8.)
- [ ] GPU artifacts (compiled kernels, ONNX/TensorRT engines,
      baked blobs) are built for the target and regenerated on it
      — not a dev-box engine committed as "the" engine. (§9.5,
      §11.1.)
- [ ] A build-time backend switch (`CUDA=1`, `USE_GPU=1`) is a
      **declared, cache-keyed build variable** (`scons gpu=1` /
      `-DUSE_CUDA=ON` / Cargo feature), not a bare
      `os.environ` read. Default is explicit and **logged at
      build time**; the chosen backend is **stamped into the
      artifact** (reportable via `--version`). Per-backend build
      dirs, not one shared `lib/`. (§9.6, anti-pattern #39.)
- [ ] A CPU↔GPU (or backend) choice is decided by **attributed**
      measurement (kernel vs host-launch vs per-frame graph
      build), and the alternative is a **runtime** switch (not a
      rebuild war). The verdict + its blocker are written down so
      it isn't re-litigated. (§9.7, anti-pattern #42.)
- [ ] Default points at the **validated** path: on hardware with
      the accelerator, the GPU path is the default *once validated
      on the target*; I did **not** revert a validated GPU stage to
      CPU without **new attributed measurement** overturning the
      record ("felt slow" / unease doesn't count). Any CPU fallback
      is a **loud, alarmed, temporary degraded mode** — never the
      silent steady state; for a safety/latency-critical stage,
      refusing-loud beats running silently slow. (§9.10,
      anti-pattern #42.)
- [ ] Build and runtime resolve the compute device from **one**
      resolver (not a SConscript arch-map vs a hardcoded
      `DEV=`), so they can't drift into a device-tagged
      artifact that JIT-mismatches the loader. Mismatch = loud
      refusal at load. Per-process GPU/CPU allocation recorded
      as policy by the process table; CPU fallback for a GPU
      daemon is a regression, not a knob. (§9.8, anti-pattern
      #43.)
- [ ] Not widening a **throughput cap** (core-pin, rate limit,
      batch size) on a shared-memory board to "use idle cores"
      without checking what it protects — it may be load-bearing,
      throttling one process to leave **memory-bandwidth** headroom
      for a latency-critical peer. The change is measured against
      the **protected** consumer's output rate, not the throughput
      of the thing sped up; load-bearing constraints carry an
      inline comment. `nice`/affinity does **not** isolate
      bandwidth contention — don't co-locate heavy work (a build)
      with the critical process; use a maintenance window. (§9.9,
      anti-pattern #44.)
- [ ] A cached / JIT-compiled accelerator path that's retained
      across calls (temporal ring, previous-frame pair, history
      window) **clones** its result before keeping it — a
      replay path reuses one output buffer, so retained entries
      alias. A single-frame test won't catch it. (§9.5 rule 7.)

## G.ter. Python environment (any dependency change)

- [ ] New / changed dependencies went in via `uv add` (or an
      explicit `uv lock`), **not** a bare `pip install` into the
      venv. `uv.lock` is staged in the same commit as the
      `pyproject.toml` change.
- [ ] `uv sync` on a clean checkout reproduces the environment I
      tested against — I didn't rely on a package that's only in
      my live venv. (See anti-pattern #22.)
- [ ] Didn't introduce a second environment manager (conda env
      file, second lock format, ad-hoc `requirements.txt`) into a
      `uv` repo. Any genuine deviation is stated in the project's
      `CLAUDE.local.md`.

## G.quint. External-service integration (talking to an API you don't control)

- [ ] Correctness has a **periodic reconcile poll** floor; any
      webhook / websocket / change-feed only *triggers an early
      reconcile* — it isn't the sole source of truth. A missed event
      is a latency blip, not permanent drift. (§25.1, anti-pattern
      #55.)
- [ ] Every write to the external system is **idempotent**: look up
      by a **stable id** (or content hash) before create (upsert);
      deletes/updates key on that id, never a mutable field. A
      failed/retried op **converges**, it doesn't duplicate. (§25.2,
      anti-pattern #54.)
- [ ] Document fields come from the **most structured source**
      (embedded PDF text, an API field) before OCR/LLM; OCR/LLM
      output is **validated** (typed amount/date parse, sanity
      checks) and falls back on low confidence. (§25.3.)
- [ ] Integration logic is testable **offline** — assert the
      request *shapes* sent (record/replay), so the suite runs
      without live credentials. (§25, §18.)
- [ ] Bulk transfers over a thin / shared link don't over-
      parallelize (concurrency is net-negative on a bandwidth-bound
      link — all time out together); timeouts are sized to
      `bytes ÷ worst-case bandwidth`, not a constant; and the
      health probe carries a **representative** payload, not a tiny
      sentinel. (§9.9, §18.1, anti-pattern #57.)

## H. Tests / how I verified

- [ ] **Diagnostic-first.** Before I changed the code, I
      produced the command that proves the cause (count this log
      line, decode this byte, list this PID, compare this inode).
      The command lives somewhere reusable, not just in this
      conversation.
- [ ] Did **not** reboot / power-cycle / restart to make a symptom
      vanish before diagnosing it. If a reset was needed, I
      **captured volatile state first** (journal/`dmesg`,
      `/proc/<pid>/wchan`+status, device + process state) to a file
      that survives it, escalated narrowest-first (handle → device →
      one daemon → **re-init the wedged subsystem's driver** →
      reboot — a warm reboot may not power-cycle a co-processor/
      SerDes, so driver re-init is often narrower *and* more
      effective), and recorded what justified a reboot. "A reboot
      fixed it" with no named cause is an open bug. (§15.7,
      anti-pattern #47.)
- [ ] For a transport/bus/feed that reads "dead": checked **both
      layers** — link up + error-free AND data actually flowing.
      A liveness check asserts on frames received, not link state.
      `link=UP frames=0` = producer-side fault. (§15.5,
      anti-pattern #32.)
- [ ] For flows that cross multiple components (target swap,
      onboard, transport rebuild), an end-to-end smoke probe
      (`scripts/_smoke_*.py`) exists and was run. The probe
      reports observed state at each step; a failed `assert`
      localises the regression to the step that broke. See
      §18.1.
- [ ] For a change with a **testable contract** (a fix, parser,
      transform, state machine): wrote the test **first** and **saw it
      fail for the right reason** (RED) before making it pass (GREEN),
      then refactored. A bug fix has a test that reproduced the bug. I
      did not ship a test I never watched fail (it proves nothing). For
      a hardware/fleet/GPU path I can't unit-test, I stated the failing
      baseline + success signal first. (§18.5, anti-pattern #78.)
- [ ] A **recovery / fallback / safe-state path** I touched is
      validated by **injecting the fault** it recovers from (kill the
      link, kidnap/teleport the estimate, truncate the file, exhaust
      the slot, SIGKILL the writer) — not just by reading the code. The
      test asserts the system **detects** (the trigger fired) *and*
      **recovers** (back to a good state), and the injection is kept as
      a regression test. The branch never runs in normal operation, so
      it and its detector rot otherwise. (§18.9, anti-pattern #23.)
- [ ] Any **mock** in my tests sits at the **slow / external boundary**
      (side effects understood), the assertions are on **real
      behaviour/output — not on the mock** being called or present, the
      mock mirrors the **complete** real structure, and I added no
      test-only methods to production classes. (§18.6, anti-pattern
      #79.)
- [ ] If my fix took **multiple attempts**: each failed attempt was
      **reverted** (not left in with another stacked on top), and if
      three evidence-based fixes failed I **stopped patching** and
      raised it as a design question instead. (§15.8.)
- [ ] Any **rate / Hz / fps readout** I added or touched **counts
      events in a window** (`N/T`) — not a smoothed reciprocal of
      inter-arrival time, which screams an impossible number on any
      burst / backlog drain and sends the operator chasing a ghost.
      (§15.9, anti-pattern #82.)
- [ ] Stated what I ran and what it told me — not just "tests
      pass." Pasted the exit code or test summary if it isn't
      obvious from context.
- [ ] If I touched a **wrapper / launcher / build / CI / scheduler
      script**, its exit code reflects the **real status of the work**
      — not a blanket 0 from a trailing `echo`, an unchecked pipe
      (`make | tee` → `tee`'s status), or a swallowed `|| true`. Used
      `set -o pipefail` / `cmd || rc=1; exit $rc`, and **tested the
      failure path** (forced the inner step to fail, confirmed it goes
      red). (§18.4, anti-pattern #77.)
- [ ] And the **complement**: a *benign* non-zero from an **auxiliary /
      optional** step (an empty glob `ls -t *.whl`, a `grep` miss, an
      **absent optional** file an `install` consumes, a best-effort probe)
      is **guarded** (`|| true`, `[ -f ]` / `[ -n "$(…)" ]`,
      consume-when-present) so it degrades instead of aborting the build /
      onboard under `set -e` — without blanket-swallowing the **essential**
      work. A shipped script and the files it references travel as one
      bundle (optional siblings in an optional file-set). (§18.4,
      anti-pattern #117.)
- [ ] If I wrote a **bootstrap / installer / scaffolder / publish**
      tool whose output a downstream tool then manages, it **finishes
      the handoff** — leaves the artifact in the state the consumer
      reads (a **committed** pin, a **pushed** tag, a **registered**
      entry), not files-written-but-uncommitted that the next tool
      can't see. The finishing commit is **path-limited** to the files
      it owns (`git add -- …`, **never `git add -A`** — WIP untouched),
      and a re-run with nothing to do is a clean no-op (idempotent).
      (§18.7, anti-pattern #87.)
- [ ] That tool's **"skip, already done" check keys on the
      consumer-visible end-state** (`local == remote` — committed **and**
      pushed; the *applied* value; the *served* artifact), **not** a
      local marker (the committed pointer alone). Otherwise a prior run
      that committed locally but **failed to push** (a flaky link) reads
      as "nothing to do" and its tail is stranded across every re-run —
      so on a failed publish, re-push the stranded commit **directly**.
      (§18.7, anti-pattern #88.)
- [ ] If I concluded something is a **floor** / "can't be
      optimized" / "won't help," I named the **mechanism** that
      makes it fundamental and the **alternative path I ruled out**
      — not an unattributed wall-clock number (which may be the
      wrong tool or a misattributed cost). (§15.6, anti-pattern
      #46.)
- [ ] If this change **reverts or re-decides something already
      settled** (CPU↔GPU, a default, a library, an architecture): I
      have **new evidence that overturns the recorded verdict** —
      not a hunch or "I'd have done it differently" — and I updated
      the record. A revert with no new evidence is re-litigation;
      a silent fallback to the abandoned path is a loud-degraded
      mode at most, never the steady state. (§18.3, recurring shape
      #6, anti-pattern #42.)
- [ ] If the test that would actually prove the change couldn't
      be run (needs real hardware, a fleet endpoint, a GPU, etc.),
      I said so explicitly. Did **not** claim success from unit
      tests alone for a change that touches hardware behaviour.
- [ ] For interactive surfaces: launched the surface and
      exercised the path. If I couldn't, said so.
- [ ] If this change is a **migration / port / rewrite**: ran a
      source-vs-target surface diff against the source at its
      **pre-migration** state (`comm -23` of public symbols), and
      every source unit is accounted for — ported, renamed (in
      the map), or on an explicit dropped-list. "The target runs"
      is not completeness. Run it before the target drifts.
      (§18.2, anti-pattern #28.)
- [ ] If the rewrite's job is to **reproduce an existing transform**
      (a bridge, decoder/codec, filter/estimator, model path, unit
      conversion): proved **behavioural parity** by dual-running the
      legacy and the new code on the **same real recorded data**
      (§16.9) and asserting equal output — bit-identical where
      deterministic, within a *stated* tolerance otherwise. "It runs"
      / "unit tests pass" is not parity. Gated cutover on it and kept
      it as a CI regression guard; until it passes the new path is
      UNVALIDATED → gated-inert. (§18.8, anti-pattern #109.)

## I. Reversibility & secrets

- [ ] No `git push --force`, no `git reset --hard`, no `rm -rf`
      against shared paths, no schema migration without a
      rollback plan — without the user explicitly OK'ing **that
      specific destructive action**.
- [ ] A **delete / cleanup / wipe** selects its targets by an
      **explicit allowlist** of the artifacts it owns (or an
      authoritative manifest), not a **broad glob** (`rm *.json`,
      `DELETE … LIKE …`) over a shared directory — a wildcard sweeps
      in co-located bystanders that merely match. (§19.1, anti-pattern
      #72.)
- [ ] A **bounded store's hard-cap eviction** (disk / ring / cache,
      "never exceed X") is enforced by a **deterministic, self-contained
      key** (oldest-first FIFO over creation time + an operator lock) —
      **not gated on a best-effort / derived signal** (upload state, a
      cache, a probe) that can be wrong. Selective "spare the un-synced
      data" logic stays in the soft / opportunistic tier. (§19.2,
      anti-pattern #74.)
- [ ] Secrets (`*.gpg`, `*_private.*`, `.lfsconfig`, signed
      bundles, fleet tokens) not read, not staged, not committed.
- [ ] **No branch left dangling by this work.** Anything I branched is
      **finished**: merged (§20.1, branch deleted, checkout returned to
      the default branch), parked with a **pushed, recorded blocker**
      (§18.3 — or landed gated-inert instead), or deleted. I didn't
      leave the checkout sitting on a feature branch whose work is
      done. (§20.3, anti-pattern #83.)
- [ ] Pushing to the right remote: ran `git remote -v` first for
      a project I hadn't pushed to before.
- [ ] Pushing to the right branch: ran
      `git symbolic-ref refs/remotes/origin/HEAD --short` if I
      wasn't sure whether the default is `main` or `master`.
- [ ] Merging to `main` on a shared checkout: only **my own
      committed + validated** branch went in — fast-forward,
      off-checkout (`git push origin <feature>:main`, no
      `--force`); I did **not** `git add -A` / commit working-tree
      WIP "to merge everything," and left collaborator branches +
      others' uncommitted files untouched. If I merged an
      **unvalidated** capability, it's **gated inert** until its
      proof lands (gate validated, feature dark, commit labelled
      UNVALIDATED) — never an unvalidated path that can act. (§20.1,
      anti-pattern #59.)
- [ ] For **concurrent / parallel / risky-isolated** work — or a repo
      another session/agent is also editing — I used a **`git
      worktree`** (own working dir per branch/task, shared `.git`) so
      trees can't collide, rather than stashing or risking an `add -A`
      of someone else's WIP. Re-init submodules in the new tree; remove
      the worktree when done. (§20.2.)
- [ ] Cutting a **release build**: committed + cleaned first — the
      build compiles the working **tree**, not git, so a dirty tree
      ships uncommitted/collaborator WIP. The artifact's stamp is
      not `-dirty`. (§20.1, §12.1.)
- [ ] A LAN-only authless service binds the **LAN interface, not
      `0.0.0.0`**, and the internet-fronted edge **actively
      rejects** its private surface (a code block, not just a
      firewall rule). Authless is only safe while the edge
      provably can't route in. (§23.1, anti-pattern #37.)
- [ ] An op through a **shared cloud edge** (heartbeat, deploy,
      upload, probe) **retries with backoff** and **rides a
      kept-alive connection** — not a one-shot short-timeout new
      connection that false-reads the edge throttling new conns as
      "peer offline." Timeout sized to the edge path, not the LAN.
      (§23.3, anti-pattern #68.)
- [ ] Reaching a **fleet host by IP** asks the **registry first** —
      its heartbeat-reported `lan_ip` (+ `last_seen`), then the reverse
      tunnel, then a direct dial to *that* IP — never a hardcoded IP
      (DHCP moves the lease) or a subnet scan (can't tell the host from
      whatever took its old address). The heartbeat reports `lan_ip` on
      **every** stage (factory + onroad), via one shared path.
      (§23.4, anti-pattern #76.)
- [ ] A **networked write that errored mid-flight** (a `push` / deploy /
      upload / API commit hit a dropped connection, lost ack, broken
      pipe, or verify timeout) is **re-verified before a blind retry** —
      the server may have applied it and lost the channel (a "failed"
      push that then says "everything up-to-date"). Read the
      consumer-visible end-state with a cheap idempotent read
      (`git fetch` + `local == remote`, a `GET`, the row's value) and
      retry only what didn't take; the write is **idempotent** so a redo
      converges, never duplicates. (§23.5, anti-pattern #91.)

### Bundled config (changes to `bake-config.py` / equivalent)

- [ ] Bake is **explicit**, never invoked automatically by
      `build.sh`, the test suite, or the runtime. The operator
      decides when to capture "my current state."
- [ ] Output file name is added to `.gitignore` *by the bake
      script itself* on first run, idempotently. Don't trust a
      manual entry to stay there.
- [ ] Runtime treats the bundle as a **seed**: schema
      validation, version check, drop-unknown-field-with-warning,
      override-invalid-with-default. The bundle cannot brick a
      fresh launch.
- [ ] Runtime **never writes** to the bundled file. Live config
      changes go to the user-home config; the bundle is
      read-only.
- [ ] Distribution channel for the bundle matches the sensitivity
      of the most secret field inside it (typically a fleet
      token). No public release page, no shared bucket without
      ACLs, no email.
- [ ] **Better: the bundle carries no secret at all.** It ships a
      checked-in, secret-free **default** (URLs, non-secret targets,
      toggles) — *not* a snapshot of the builder's `~/.config` — and
      the credential is **seeded per-device at runtime** (the
      operator's own key / provisioning / tokenless audited
      enrollment, §26.4). A bake/build step **greps the artifact and
      refuses to ship** if any token/key/password field is present. A
      distributable artifact is a copy machine — a baked shared token
      leaks to every recipient. (§14.5, anti-pattern #113.)

---

## Quick greps cheat sheet

```sh
# Generic service-layer rule: no widget-class imports below the GUI boundary
git diff --cached -- 'app/**' 'services/**' 2>/dev/null | grep -n 'QtWidgets'

# "Default ON" Params trap — naive None check
git diff --cached | grep -nE 'get_bool\(.*\).*is\s+None'

# Speculative TODOs about to ship
git diff --cached | grep -nE '\b(TODO|FIXME|XXX)\b'

# Secrets accidentally staged
git diff --cached --name-only | grep -nE '\.gpg$|_private\.|\.lfsconfig$|\.zip$'

# Forgotten print / breakpoint
git diff --cached | grep -nE '^\+.*\b(breakpoint|pdb\.set_trace|print\(|console\.log)\b'

# Mode-label collision
git diff --cached | grep -niE '"debug mode"|'\''debug mode'\'

# Silent-default schema trap — getattr with falsy fallback
git diff --cached | grep -nE 'getattr\([^,]+, *["'\''][^"'\'']+["'\''], *(0|False|None|""|b"")\)'

# Per-tick log inside a loop (heuristic — false positives normal,
# look at each hit and confirm it's transition+heartbeat shaped)
git diff --cached | grep -nE '^\+.*\b(while|for)\b.*:' -A8 | grep -nE 'log\.(info|debug|warning)|print\('
```

If any of those return a hit, stop and look at it.
