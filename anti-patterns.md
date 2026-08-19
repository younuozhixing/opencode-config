# Anti-patterns

Bugs we've already shipped, reframed so the lesson transfers. Each
entry is short on purpose: symptom → looks like → actually is →
fix. Three lines beats ten — the point is fast recognition next
time, not narrative.

The entries are project-neutral. They name the *shape* of the bug,
not the specific service or file. When you recognise the shape,
the fix below transfers without translation.

<a id="recurring-shapes"></a>
## Recurring shapes — most of these are one of six cycles

The 69 entries below are not 69 unrelated bugs. They are mostly
**re-runs of six root shapes** in different layers. If you find
yourself debugging something that rhymes with a past bug, you're
probably inside one of these — fix the *class*, not just today's
instance.

1. **Silent divergence: authoritative vs. actual.** A source of
   truth and a live copy drift apart with nothing making the gap
   loud, so each runs on a different reality.
   → #2 (config default vs fresh install) · #22 (deps vs lock) ·
   #25 (write lands where the reader doesn't look) · #26 (layout
   code vs persisted cache) · #29 (build vs deployed rig) ·
   #31 (declared services vs running) · #39 (build-host env forks
   the artifact) · #41 (mirrored peer-version constant) · #43 (build-tagged device
   vs runtime device) · #53 (dev source layout vs compiled/installed
   deployment) · #55 (local state vs an external service when a
   pushed event was missed) · #67 (a green-once build vs a
   reproducible one) · #74 (a hard cap gated on a flaky signal that
   diverged from the truth) · #76 (a hardcoded / scanned IP vs the
   registry's self-reported `lan_ip`) · #80 (the deployed box drifted
   AHEAD of the repo — the next deploy clobbered the live fix) · #83
   (the fix lived on a branch nobody merged; the checkout parked
   there) · #84 (`--clean` left stale state — wipe and build disagreed
   where it lives). *Break
   it:* make the gap a command you can run, and let the authoritative
   side win. → also #69 (two
   copies of critical-path logic drift; factor to one shared module).

2. **"Looks healthy" ≠ "is working."** A check passes on a proxy
   (link up, process alive, total flowing, badge green) while the
   real thing is dead.
   → #1 (UI subscribed, bridge not forwarding) · #17 (daemon
   runs, publishes nothing) · #23 (recovery path is dead code) ·
   #32 (link up, no data) · #33 (aggregate green, one element
   dead) · #34 (always-red signal nobody reads) · #38 (GPU
   alive but fence-blocked) · #40 ("GPU" work silently on CPU,
   "loaded" hiding 0 Hz) · #48 (consumer alive, publishes 0 —
   inputs on different clocks) · #56 (web app green at localhost
   root, 404s under the proxy prefix) · #57 (tiny-sentinel probe
   green while real multi-MB transfers 502) · #58 (single-threshold
   gate passed corrupt input; "deployed" ≠ applied) · #60 (UI looked
   fine; the server trusted a client-posted value) · #62 (whole
   stack "frozen" — one client leaked a capped shared resource) ·
   #63 (topic alive at producer, silent to new subscribers — IPC
   reaped on logout) · #64 (consumer drops a fast stream, one read
   per tick) · #65 (adaptive lever re-inited shared HW live, wedged
   it under load — "verified standalone" at idle missed it) · #68
   (peer "offline" — really the shared edge throttled a one-shot
   new connection) · #73 (consumer pinned to a dead session's pushed
   override while the producer is healthy) · #75 (control pill showed
   "engaged" on write-OK; the target rejected it same-tick) · #77 (a
   wrapper exited 0 while the work failed — green CI/scheduler on a
   lie) · #78 (a test never seen red — green that can't fail proves
   nothing) · #79 (a test asserting on its own mock — green while the
   real integration is broken) · #82 (a rate meter screaming an
   impossible number — the inverse: looks-broken ≠ broken, the meter
   lied). *Break it:* assert on the specific
   thing crossing, never the proxy.

3. **Silent-falsy / wrong-default.** A missing value comes back
   as a plausible falsy default instead of an error, and the
   wrong branch is taken quietly.
   → #2 · #16 (`getattr(…, default)` on a moved field) ·
   #18 (threshold-pair drift) · #39 (unset build flag →
   silent CPU fallback) · #70 (a ported stack's inherited hardware
   constants — upstream's device, not yours) · #81 (a missed wiring
   line silently selected the not-inert base class — publisher
   collision kills the victim). *Break it:* distinguish
   *absent* from *false* at the boundary; fail loud on absent.

4. **Wrong-platform / wrong-form artifact.** A binary or file
   correct for one platform/context reaches another where it's
   silently invalid.
   → #4 (rsync clobbers target-arch binary) · #27 (consumer
   builds against a source tree) · #36 (wrong-platform `.so`
   linked) · #50 (freezer silently drops a module whose native dep
   was absent at bundle time). *Break it:* select the artifact by
   `(os, arch)` / pin a published version; verify by importing /
   running it, don't assume.

5. **Always-asserting carries no information.** A signal emitted
   every tick / always-on trains everyone to ignore it, and
   starves the channel it shares.
   → #19 (per-tick log starves the loop) · #34 (perpetually-red
   check). *Break it:* transition + heartbeat, rate-limit,
   green-in-good-state-or-delete.

6. **Decision oscillation: a settled choice keeps getting
   re-litigated.** A decision — CPU vs GPU, which library, a
   default value, an architecture — is made, reverted, re-made,
   burning days, because it was never *recorded as settled with the
   evidence that settles it*. So the next person (or you, next week)
   re-derives it from scratch — usually from a hunch or an
   un-attributed number — and swings it back.
   → #42 (CPU↔GPU flip-flop) · #46 (an unattributed "floor" that
   freezes then unfreezes the work) · #47 (a reboot that "fixes" it,
   so the real decision is never made) · #49 (blind-reverting the
   loudest/newest suspect instead of attributing the cause).
   *Break it:* decide once by
   attributed measurement, **record the verdict + its blocker + the
   evidence-bar to re-open**, and let the recorded default win until
   *new evidence* overturns it — not unease
   ([§18.3](delivery-rules.md#decision-record)).

Recognising the cycle is the leverage. The per-entry fixes below
are the same six moves applied to a specific layer.

---

## 1. UI panel dead / stuck because the bridge isn't forwarding its topic

**Symptom.** Operator triggers a transition; the UI advances to a
"…ing" state and never reaches the terminal state, even though the
backend is healthy and emitting.

**Looks like.** A UI parser bug. A field name typo. A race
between the trigger and the first response.

**Actually is.** The transport bridge between backend and UI is
not forwarding the topic the state machine waits on. The UI is
subscribed correctly; nothing is being delivered.

**Fix.** Inspect the bridge's outbound topic list, not the UI. Add
the missing topic to the bridge, restart the bridge. The terminal
state arrives.

---

## 2. "Default ON" feature off on fresh install

**Symptom.** A feature documented as default-on is off on a fresh
install. After someone toggles it once it works forever.

**Looks like.** A typo in the config key. A wrong default in the
config table.

**Actually is.** The KV config store returns **falsy** (`False`,
empty bytes, empty string) for an unset key — not `None`. The
naive

```python
raw = config.get_bool("UseFeature")
if raw is None:        # never fires on a fresh install
    return True        # documented default
return raw
```

silently falls through to `bool(False)` on every fresh install.

**Fix.** Accept only explicit opt-out values:

```python
def use_feature() -> bool:
    env = os.getenv("UNO_FEATURE")
    if env is not None and env.strip().lower() in ("0", "false"):
        return False
    raw = config.get("UseFeature")        # bytes / str / None
    if raw in (b"0", b"false", "0", "false"):
        return False
    return True
```

Apply every time you add a "default ON" config bool. Document the
explicit opt-out values inline.

---

## 3. `widget.close()` from a worker thread → macOS deadlock

**Symptom.** The interactive surface hangs on macOS during a long
operation (onboarding, batch import, download). The progress
dialog never closes.

**Looks like.** A slow remote call. A blocking subprocess. The
asyncio loop pinned.

**Actually is.** A worker thread (or a `concurrent.futures`
callback) calls `dialog.close()` or `widget.deleteLater()`
directly. On macOS this deadlocks because the Qt main loop owns
the widget tree.

**Fix.** Canonical async progress dialog:

```python
prog = QProgressDialog(...)
prog.setModal(True)

def _finished(ok: bool, msg: str):
    QMetaObject.invokeMethod(
        prog, "done", Qt.QueuedConnection,
        Q_ARG(int, 0 if ok else 1),
    )
worker.finished.connect(_finished)
# hard deadline so a stuck worker doesn't trap the operator
QTimer.singleShot(60_000, lambda: prog.done(2))
prog.exec()        # modal — drives the event loop
```

Don't `processEvents()` spin-wait. Don't close from the worker
callback. Queue the close back to the GUI thread.

---

## 4. Host-arch binary clobbered the target-arch binary on sync

**Symptom.** After a routine host→target rsync the daemon shows no
output (no frames, no packets, no responses). The systemd log
shows the daemon starting without errors.

**Looks like.** A runtime pipeline bug. An accelerator out of
memory. The IPC server not reachable.

**Actually is.** The host-built (e.g. x86_64) native binary in the
source tree got copied over the target-built (e.g. ARM64) one. The
binary fails to exec the underlying pipeline; the daemon runs but
publishes nothing.

**Fix.** Exclude the native binary path explicitly:

```sh
rsync -avz \
    --exclude='*.o' --exclude='*.os' \
    --exclude='__pycache__' --exclude='.venv' \
    --exclude='<path/to/each/native-arch binary>' \
    ./ <user>@<target>:<remote-path>/
```

Belongs in muscle memory. Most native binaries have no extension,
so `*.bin` will not catch them — list each path.

---

## 5. Blocking the async event loop — CPU work *or* a slow synchronous call — stalls every coroutine

**Symptom.** Traffic on a shared event loop freezes (telemetry at a
fraction of rate, liveness chips going stale) whenever some work
runs. Worse: connections that depend on loop-driven timers
**collapse** — a WebRTC/RTC session opens then drops a few seconds
later (ICE/DTLS/SCTP keepalives missed), a heartbeat peer marks the
link dead, reconnect logic double-fires.

**Looks like.** Slow producers upstream. Network renegotiation.
A flaky peer. Bad client CPU.

**Actually is.** Something ran *inline* on the event loop and held
it for too long — either **CPU-bound** work (image decode, multi-MB
JSON parse, a big hash) *or* a **slow synchronous/blocking call**: a
sync network recv, a DNS / name / topic resolve, a subprocess wait,
`time.sleep`, any >~100 ms blocking I/O. The loop can't service its
other coroutines *or its own protocol timers* for the duration — so
co-resident subscribers stall and timer-driven connections drop. A
synchronous "just measure it real quick" probe (e.g. ~1.2 s of
blocking recvs) inside a connection handler is the classic trigger.

**Fix.** Keep the loop non-blocking. For heavy CPU, offload to an
executor:

```python
result = await loop.run_in_executor(None, heavy_sync_call)
```

For a value the loop needs *synchronously* on a hot path (a rate, a
resolved address), don't probe inline — run the blocking measure in
a **daemon thread behind a background-refreshed cache**, and have
the loop **read the cache** (sub-ms), defaulting to a safe value
until the first measurement lands. Never put a sync recv / resolve /
subprocess on the loop; if you need a blocking loop, give it its own
thread and pipe results back via a queue.

---

## 6. Conflating four superficially similar identifiers

**Symptom.** A consumer opens the requested artifact, then dies
with "not found." A sibling consumer with the same input works.
Or vice versa.

**Looks like.** A tool bug. A bad cache. A storage-layer issue.

**Actually is.** A family of identifiers that *look* the same but
mean different things got passed to the wrong consumer. Example
shape:

| Variable | Form | Used by |
|---|---|---|
| `*_id` | `ns\|key` | one tool |
| `*_basename` | bare basename | another tool |
| `*_dirname` | on-disk directory name | the logger |
| `*_key_prefix` | object-store prefix | the storage layer |

**Fix.** Don't reuse one variable name when you mean a different
form. Don't `basename(x)` in passing — that's the bug. Define one
name per form and pass the right one to the right consumer.

---

## 7. Side-channel "works" but the canonical entry point doesn't

**Symptom.** Operator pushes state via an ad-hoc SSH script; it
works. Same state via the canonical onboarding / bootstrap flow
fails or silently no-ops.

**Looks like.** A flow bug. A timeout. A parse error in the
canonical path.

**Actually is.** The side-channel skips the bootstrap sequence
(installs, restarts, config writes, signal-bus rewiring) that the
canonical flow performs. The state is half-applied; the system
appears to "work" because the operator's mental model accidentally
fills in the gap.

**Fix.** Reproduce the failure via the canonical flow. The bug is
in the canonical flow (or in its bootstrap script). **Don't ship
"push by SSH" as a workaround** — that bakes in a permanent
divergence the next operator will trip on.

---

## 8. Assumed default branch name

**Symptom.** `git pull --ff-only origin main` fails: "couldn't
find remote ref main."

**Looks like.** Bad remote URL. Stale clone.

**Actually is.** Not every repo's default branch is `main`. Some
older repos default to `master`.

**Fix.** Read the default first:

```sh
git symbolic-ref refs/remotes/origin/HEAD --short
# or
git remote show origin | grep 'HEAD branch'
```

Don't pattern-match `main` (or `master`) across siblings.

---

## 9. Wire-format compatibility creep

**Symptom.** A PR proposes "improving" a deliberately standalone
component by making its identifiers, framing, or magic numbers
match an adjacent stack's, citing "compat" or "less divergence."

**Looks like.** A reasonable cleanup. The absence of compat looks
like a debt waiting to be repaid.

**Actually is.** The absence of cross-stack interop is a deliberate
*property* of the design — it removed an entire class of
divergence-debugging surfaces. Re-introducing wire-compat doesn't
unlock anything currently used; it just reverses the boundary and
re-introduces the surface the boundary was drawn to eliminate.

**Fix.** Don't. If wire interop is actually needed, build the
bridge at the application layer (translating in/out at the edge),
not by collapsing the wire formats.

---

## 10. Cloud edge for off-LAN access vs. reverse tunnel

**Symptom.** "I can't reach the LAN-side service from outside the
office." Someone adds a vhost on the company's cloud nginx /
WordPress edge.

**Looks like.** Plausibly the right fix.

**Actually is.** The cloud edge is network plumbing — it routes
traffic. It is not the front door for the LAN-side service. The
intended off-LAN access pattern is a **reverse tunnel** from a
cloud edge port to the LAN-side port. Cloud security-group rules
are narrow on purpose; widening them for a one-off creates a
permanent attack surface.

**Fix.** Use (or extend) the existing tunnel service. Don't add a
cloud vhost; don't widen the security group.

---

## 11. Naming-drifted cancellation protocols across workers

**Symptom.** A dialog's `closeEvent` (or equivalent teardown hook)
ends up doing string introspection on worker objects —
`getattr(worker, 'stop', None)`, iterating signal names, calling
whichever cancel method is spelled this time.

**Looks like.** Defensive coding. Backward compatibility.

**Actually is.** Workers across the project each invented their
own cancel primitive (`_stop` vs `_cancel` vs `_abort` vs
`finished` vs `requestInterruption`). The introspection is the
cost of a missing unified contract.

**Fix.** Pick one cancellation primitive — a `threading.Event`, a
single `stop()` method, a registry hook — and impose it on every
worker via a base class or a registration step. If the upstream
class hierarchy makes that hard, fix the hierarchy. The "many
shapes glued together by introspection" pattern always loses
under load (a new worker arrives whose shape you don't catch).

---

## 12. Per-event publish under load

**Symptom.** A streaming-log dialog stutters or drops events at
30 lines/s even though the producer subprocess isn't pegged.
Profilers show snapshot-lock contention.

**Looks like.** UI render too slow. Lock too coarse. Disk too slow.

**Actually is.** A per-event publish path: every appended line
copies the buffer (`tuple(deque)` is O(N)), takes the snapshot
lock, wakes the UI. At 30 events/s on a 2000-line buffer that's
60k element copies and 30 lock cycles per second per producer.
Add a second producer and the lock starves.

**Fix.** Coalesce publishes: ship a new snapshot at the smaller
of N events or T milliseconds:

```python
_PUBLISH_BATCH_N = 10
_PUBLISH_INTERVAL_S = 0.05    # 50 ms — below operator perception
```

The intermediate buffer (`collections.deque(maxlen=N)`) absorbs
events between snapshots. 50 ms is below the operator perception
threshold (~100 ms), so the UI still feels real-time.

---

## 13. Unbalanced framework stack on exception

**Symptom.** An exception inside a per-frame UI callback (a video
tile, a panel, a dialog) takes the **whole UI down** with an
assert like `Missing EndGroup()` or `ImGui assertion failure`.

**Looks like.** A framework bug. Renderer corrupt state.

**Actually is.** Immediate-mode UI APIs use begin/end pairs
(`begin_window` / `end`, `push_id` / `pop_id`, `begin_group` /
`end_group`). An exception thrown between a `begin` and its
matching `end` leaves the framework's internal stack unbalanced.
The next frame's first `begin` (or shutdown's final `end_frame`)
asserts.

**Fix.** Wrap top-level draw functions with a `@safe_view`-style
decorator that **catches the exception AND unwinds the begin/end
stack back to the depth the view started at**. The robust way is
the framework's **own error-recovery**: snapshot its stack-size /
error-recovery state *before* the view and, on exception, call its
recover-to-snapshot routine (Dear ImGui's
`error_recovery_try_to_recover_state`) to emit the matching
`End`/`EndChild`/`Pop*`. That covers *every* open scope — including
the ones a wrapper can't see from outside. Disable the framework's
recover-time assert (`config_error_recovery_enable_assert`) so
recovery doesn't itself abort. Where no such mechanism exists, fall
back to `try`/`finally` at each call site:

```python
imgui.begin_group()
try:
    risky_draw()
finally:
    imgui.end_group()
```

Rate-limit the error log (once per N seconds per view) so a
per-frame exception doesn't drown the journal.

---

## 14. Input listener consumes keys while a text field has focus

**Symptom.** Operator types into a filter / search / chat field;
the robot lurches. A "w" in the search box jammed throttle.

**Looks like.** A driving-input bug. Joystick state corruption.
An overzealous keyboard hook.

**Actually is.** A driving-input listener (WASD for
throttle/steering, keyboard fallback for gamepad axes) consumes
keys without checking whether some other widget is already
focused on text input.

**Fix.** Guard every input consumer with the framework's "is some
widget already capturing this input?" predicate **before** the
input-to-actuator path runs:

```python
io = imgui.get_io()
if io.want_capture_keyboard:
    return                      # text widget owns the key
# else: consume for driving control
```

This is a **safety rule**, not a UX one. Treat regressions here
with the same gravity as widening an actuator limit.

---

## 15. Per-tick allocator in a tight loop

**Symptom.** RSS slowly oscillates upward at run time; tracemalloc
top-growers points at one line that allocates a fresh capnp /
protobuf / dataclass instance every loop tick. GC catches up
eventually so it's not an unbounded leak — but the working set
sits 20–40 MB above what it could.

**Looks like.** A memory leak. A library bug.

**Actually is.** The tight loop (driving-input publisher, 100 Hz
sensor tick, render tick) calls `lib.new_message(...)` or
`SomeStruct()` per iteration. Each instance is reclaimed by GC,
so there's no *unbounded* growth — but the per-tick churn is
visible in tracemalloc and wastes allocator throughput.

**Fix.** Pre-build the builder / struct once outside the loop and
reuse it. Rust-rewriting the allocator is the wrong altitude for
this — it's 5 lines of orchestration:

```python
# before:
def tick():
    msg = lib.new_message("foo")
    msg.foo.value = read_value()
    publish(msg)

# after:
_msg = lib.new_message("foo")            # built once
def tick():
    _msg.foo.value = read_value()
    publish(_msg)
```

Verify with a memory-probe script showing the line drop out of
the top-growers report.

---

## 16. Schema misread: `getattr(obj, name, default)` against a moved field

**Symptom.** A downstream pipeline silently behaves as if a flag
is never set. The producer log says the flag toggles; the
consumer treats every event as "flag clear." Hours / days later
the consequence shows (e.g. a keyframe-priority queue drops
every IDR because `is_keyframe` is perpetually `False`).

**Looks like.** A producer-side bug. A timing race. A serialiser
that elides the field.

**Actually is.** The schema moved the field — top-level
`evta.flags` became nested `evta.idx.flags` in a version bump —
and the consumer reads it as:

```python
flags = getattr(evta, "flags", 0)        # silently 0 forever
```

The fallback default looks like a safety net. It's a silent
miss. `evta.flags` genuinely doesn't exist; `getattr` returns
the falsy default; downstream sees `0` and is none the wiser.

**Fix.** Don't `getattr(..., default)` against a schema you
control. Read the field directly and let `AttributeError` at the
call site tell you the schema moved. If you must use `getattr`
(across versions, across vendors), make the default a non-falsy
sentinel and raise on it:

```python
_MISSING = object()
raw = getattr(evta, "flags", _MISSING)
if raw is _MISSING:
    raise SchemaMismatch("evta.flags moved — check evta.idx.flags?")
```

---

## 17. Daemon "runs" but publishes nothing (silent permission / resource failure)

**Symptom.** A daemon is listed as healthy by the supervisor and
`pgrep` shows it alive. But the topic it should publish is
empty; the consumer eventually fires "data invalid." The
operator first suspects the consumer or the wire.

**Looks like.** Subscriber bug. Wire down. Message conflation
dropping everything.

**Actually is.** The daemon opened a transient resource at
startup (`/dev/ttyACM*`, a CAN socket, a GPU context, a TCP
connect), hit a `PermissionError` / `FileNotFoundError` /
`ConnectionRefusedError`, **caught the error and slept silently
forever** instead of retrying with a warning. The daemon is
running; the workload isn't. Often combined with the
supervisor's `restart_if_crash=False` — one-shot startup error
strands the daemon for the whole boot.

**Fix.** Retry with a loud WARNING inside the daemon:

```python
while not shutdown.is_set():
    try:
        fd = open(resource_path, "rb")
        break
    except (PermissionError, FileNotFoundError, OSError) as e:
        log.warning("opening %s: %s; retrying in 5 s", resource_path, e)
        shutdown.wait(5.0)
```

Plus a snapshot field that exposes the failure mode (`live=False`,
`error=str(e)`) so operators see "not yet ready" instead of
"publishing zero frames." See [§6 Process supervision](engineering-rules.md#process-supervision) "Transient-resource daemons" and [§16.4](delivery-rules.md#live-field-convention).

**Common shape that hits this:** a wrapper exception like
`pyserial.SerialException` that *wraps* `PermissionError`. A bare
`except PermissionError` doesn't catch it; the daemon dies anyway.
Match on the inner message
(`"Permission denied" in str(exc)`) or catch the wrapper class.

---

## 18. Threshold-pair drift causing flicker

**Symptom.** A gated output (joystick mode, engage state, alert
banner) flickers off briefly under normal jitter, then back on.
Operator reports *"the mode keeps dropping"*; the bug looks like
the worker is unreliable.

**Looks like.** Network jitter. Subscriber timeout too short. A
race in the producer.

**Actually is.** Two constants that gate the same condition were
defined separately and drifted apart over edits. A "stale input"
gate at the top of the loop says `> 0.2`; an "anti-flicker hold"
50 lines below says `HOLD_SECS = 0.5`. A 200–500 ms gap trips
the stale gate, short-circuits the held grace, and drops the
output for one cycle. The held grace never saves anything
because the upstream gate fires first.

**Fix.** **One constant per gate**, named, referenced
everywhere:

```python
HOLD_SECS = 0.5

def tick(now, last_recv_t):
    stale = (now - last_recv_t) > HOLD_SECS    # same threshold
    if stale and not held:
        drop_output()
```

When you change the threshold, you change one number. If you
find yourself writing the literal `0.5` twice in the same file,
that's a refactor cue, not a typing accident to keep both.

---

## 19. Per-tick log line starves the event loop

**Symptom.** A long-running interactive surface degrades over
hours — video stutters, status traffic stops, reconnects
double-fire. Restarting fixes it instantly. RSS is climbing
slowly but not pathologically.

**Looks like.** Memory leak. Slow consumer. Network reconvergence
needed.

**Actually is.** A log line in an inactive code path that fires
on **every** producer tick — e.g. `_run()` checks `_active`
*after* it logged "first frame received," then resets the seen
flag so the next tick logs again. At 10 Hz × 6 inactive subjects
that's 60 lines/s pouring into the journal. The asyncio writer
serialises log emissions; aiortc DC `send()` calls wait behind
the log; outgoing bufferedAmount fills; active subjects degrade.

**Diagnose:**

```sh
journalctl -u <service> --since "60s ago" | grep -c "<spam phrase>"
# >20 per 60 s on an idle-ish system = bug active
```

**Fix.** Check the inactive predicate at the **top** of the
loop. Don't fall through to the "first time seen" log when
inactive. See [§16.3](delivery-rules.md#log-spam-starves-loop):

```python
async def _run(self):
    while not self._stop.is_set():
        if not self._active:
            await asyncio.sleep(0.05)
            continue
        msg = await self._recv()
        if not self._first_seen:
            log.info("first frame after %.3fs", elapsed)
            self._first_seen = True
        ...
```

**Generalisable rule:** any unconditional `log.*` inside a tick
loop is a performance bug. Transition + heartbeat ([§16.1](delivery-rules.md#transition-heartbeat)) is the
shape, always.

---

## 20. Defensive assert on a transient race aborts instead of retrying

**Symptom.** A fresh subscriber crashes ~15 s after attaching to
a high-rate publisher on a specific platform (often ARM). On
other platforms the same subscriber is fine. The error is a
`assert(size < q->size)` or similar sanity check in C code.

**Looks like.** A real data-corruption bug. Hardware issue.
Driver problem.

**Actually is.** A defensive sanity check fires on a brief
**legitimate** window: under weak memory ordering, the reader
catches the queue header between the writer's pointer update and
its size-field update. The cast reads garbage, exceeds the
"impossibly large" threshold, the assert aborts. Next loop iter
would have seen the real value.

**Fix.** A defensive check on a value that can be transiently
torn under your memory model should **retry**, not **abort**:

```c
size = *size_p;
if ((uint64_t)size >= q->size) {
    // transient torn read under weak ordering; try again
    continue;
}
```

Aborting on a transient condition is the bug. The check exists
to catch corruption, not to catch the producer mid-write.

**Generalisable rule:** before changing an assert to a retry,
prove the condition is genuinely transient (e.g. the writer
publishes a valid value within bounded time). If it isn't —
e.g. the value is logically derived from independent state — a
retry just hides the real bug.

---

## 21. UI reflects local input, not the value going on the wire

**Symptom.** Operator pushes the joystick forward; the on-screen
crosshair moves; the vehicle doesn't. Or: gamepad disconnects;
the publisher correctly zeroes the axes; the on-screen crosshair
still shows the last typed values. Or: WASD path works in dev;
the merged-with-gamepad publish loop has a bug; the GUI looks
fine.

**Looks like.** Publisher / wire / bridge bug. Subscriber
ignoring updates. Driver issue.

**Actually is.** The control widget draws from **its own local
read** (the WASD keys it just received, the gamepad poll it just
did) instead of from the **publisher's snapshot of what's
actually on the wire**. The widget shows the operator's intent;
the publish loop may be sending something completely different
(zeroed by a watchdog, merged with another source, blocked by
the channel being closed). The operator sees a UI that says
"throttle 0.4" and a vehicle that says "throttle 0."

**Fix.** Read from the publisher's snapshot, not the local input
state:

```python
# WRONG
def draw_joystick(self, local_keys):
    crosshair.draw(local_keys.x, local_keys.y)

# RIGHT
def draw_joystick(self, joystick_service):
    snap = joystick_service.snapshot()
    crosshair.draw(snap.axes[0], snap.axes[1])
    if snap.stale:
        imgui.text_colored(ERR, "wire silent — axes zeroed")
    imgui.text(f"tx {snap.n_sent} frames")
```

The widget gains three things:

1. It reflects the merged source (WASD + gamepad + web client).
2. It reflects the watchdog's zeroing if the wire went silent.
3. It exposes a frame counter — operators (and the assistant
   debugging on their behalf) immediately see "we are sending"
   vs "we are NOT sending."

See [§8.19](ui-rules.md#wire-state-mirror) for the rule and [§8.21](ui-rules.md#fault-localization) for the broader fault-
localization diagnostics pattern.

---

## 22. "Works on my machine" — Python dependency drift

**Symptom.** A feature works for the author. The teammate, CI, or
the embedded target hits `ModuleNotFoundError`, a version
mismatch, or subtly different behaviour. `uv sync` on a clean
machine doesn't produce the environment the author is running.

**Looks like.** A flaky import. A platform difference. A missing
system library.

**Actually is.** The author `pip install`ed a package directly
into the venv. It works locally because the live environment has
it — but the install never touched `uv.lock`, so the lock and the
real environment have **silently diverged**. Anyone who builds
the environment from the lock (every other machine) gets the old
graph.

**Fix.** The lock is the source of truth. Add dependencies through
the lock, never around it:

```sh
# WRONG — mutates the venv, leaves the lock stale
.venv/bin/pip install some-package

# RIGHT — updates pyproject.toml AND uv.lock atomically
uv add some-package
git add pyproject.toml uv.lock
git commit -m "deps: add some-package"
```

To recover an already-drifted repo: `uv sync` to reset the venv
to the lock, confirm the feature breaks (proving the dep was
never locked), then `uv add` it properly. See
[§20](delivery-rules.md#python-env).

**Generalisable rule:** whenever the live state and the committed
state of an environment can diverge silently, the committed state
must be the one you mutate. Lock, commit, push — then it's
portable.

---

## 23. The recovery path you trust is dead code

**Symptom.** A degraded element (one stalled stream, one wedged
connection) never recovers, even though there's a recovery
mechanism in the code that's *supposed* to handle exactly this.
The operator believes the system self-heals; it doesn't.

**Looks like.** The recovery mechanism has a bug. A threshold is
wrong. A race.

**Actually is.** The recovery mechanism **never runs** — it's
dead code. The trigger it waits on is never emitted (a protocol
the transport doesn't actually use anymore, a callback that was
unwired in a refactor, an event the upstream stopped sending).
The mechanism looks alive in the source; it's a corpse at
runtime. Example: a per-track keyframe-request recovery that
assumes a video path which is actually DataChannel-only — the
request goes nowhere, the recovery never fires.

**Fix.** Two steps:

1. **Confirm the trigger fires.** Add a log/counter at the entry
   of the recovery path and verify it increments under the
   failure. If it never increments, the path is dead — the bug
   is "recovery doesn't run," not "recovery is buggy."
2. **Build the recovery on a trigger that actually exists.** For
   the stalled-stream case, a *stall watchdog* (last-frame age
   exceeds threshold → bounce the connection, capped) runs on
   data the transport genuinely produces, instead of waiting on
   a protocol event that never comes. See
   [§10.2 capped recovery](engineering-rules.md#capped-recovery).

**Generalisable rule:** a recovery / failover / retry mechanism
is only real if its trigger is on the live path. Before trusting
one, prove the trigger fires. "The code is there" is not "the
code runs."

---

## 24. Raw infrastructure detail or sensitive data on an always-visible surface

**Symptom.** A demo, screen-share, or bug-report screenshot leaks
the fleet's network map — public edge IP, internal target
addresses, ports, `user@host` — or someone's **bank card / ID /
phone / API key** sitting in a list. Discovered after a customer
call, not before.

**Looks like.** Not a bug at all until someone notices the leak.

**Actually is.** A status banner / panel / list that's **always on
screen** renders raw infra detail or sensitive user data because it
was the easiest thing to draw. Always-visible means
always-in-screenshots; the passive surface became an exfiltration
channel nobody designed.

**Fix.** Always-visible surfaces show a **masked / friendly** form
(hostname, role label, masked IP, `•••• 4321`); the raw detail
lives in a hover tooltip or behind an explicit "reveal" the user
takes deliberately. Sensitive data is masked **by default**;
revealing it is the action, never the resting state. See
[§8.22 masking](ui-rules.md#masking).

```python
# WRONG — banner is always on screen, always in screenshots
banner.text(f"{lan_ip} → {vps_public_ip}")
# RIGHT — shape on the banner, detail behind a hover
banner.text(route_label)              # "LAN" / "TUNNEL"
on_hover(lambda: tooltip(f"{lan_ip} → {vps_public_ip}"))
```

This is distinct from "don't commit secrets" (that's the repo;
this is the screen). Both are leak-prevention; the surface
differs.

---

## 25. Write "succeeds" into a path the reader never reads

**Symptom.** A setting is written, the write returns success, the
UI shows a ✓ — and the behaviour doesn't change. A flag set to
enable a feature leaves the feature off. The on-screen manual
toggle for the *same* setting works; the scripted write doesn't.

**Looks like.** The write failed silently. A permissions issue.
The reader caching a stale value.

**Actually is.** The writer and the reader resolved **different
paths** for the same logical key. A bootstrap script hardcoded a
platform-default location (`/data/params`, `/etc/app`, a
well-known absolute path) that's correct on one platform and
wrong on the one it ran on. The write lands in a real directory —
so it *succeeds* — but the reader (which computes the path from
the platform / env / API) looks somewhere else. The data sits in
an orphan directory forever.

The on-screen toggle works precisely because it goes through the
**same API the reader uses**, so it resolves the same path.

**Fix.** Never hardcode a platform-specific storage path in a
script. Go through the same API/resolver the application uses:

```python
# WRONG — hardcoded path, correct on one platform only
echo 1 > /data/params/FeatureEnabled

# WRONG — same path assumption in Python
open("/data/params/FeatureEnabled", "w").write("1")

# RIGHT — the API resolves the same root the reader will use
Params().put_bool("FeatureEnabled", True)   # run AS the run-user,
                                            # inheriting its HOME/env
```

If the path is genuinely env-derived, run the writer **as the
same user with the same environment** the reader runs under
(`sudo -H` to inherit HOME, no override of the path env var) so
both resolve identically.

**Generalisable rule:** a write is only real if it lands where
the reader looks. When writer and reader can disagree on
location, make them share one resolver — never two copies of the
path logic that can drift. Same family as anti-pattern #2 and
the silent-default traps: the failure is silent because the write
half *succeeded*.

---

## 26. Layout overlap that keeps coming back

**Symptom.** A UI layout overlap — bottom status bar over the
docks, two panels colliding, a banner clipping content — gets
fixed, then reappears days later. Fixed again, back again. The
team starts to believe the layout engine is flaky.

**Looks like.** A framework docking bug. A race in layout init.
A DPI rounding problem.

**Actually is.** The "fix" was a **hand-drag in a running build**,
which the framework wrote into the *persisted* layout cache
(`imgui.ini` and equivalents). It looked fixed on the fixer's
machine because their cache now holds the dragged arrangement.
But the **programmatic** layout (the dock-split code) still
overlaps, and that's what every machine with no cache — and,
because programmatic splits usually apply `first_use_ever`, every
*future* relaunch on a wiped cache — actually uses. Cache and
code diverged; whichever you didn't fix is what the next person
sees.

A second trigger of the same cycle: a font / metric change
without bumping the layout cache key ([§8.23](ui-rules.md#device-sizing)) —
old splits computed for old metrics get restored over new
content.

**Fix.**
1. Fix overlap in the **layout code** — reserve the status-bar /
   banner height in the dockspace, set the splits programmatically
   — not by dragging.
2. **Bump the layout cache key** so existing caches are discarded
   and the code layout takes effect for everyone.
3. **Verify cold:** delete the persisted layout file, relaunch,
   confirm no overlap from that fresh state.

**The one test that ends the back-and-forth:** "would a
brand-new machine with no cache file see this fix?" If you can't
answer a confident yes, you changed the cache, not the code, and
it *will* come back.

**Generalisable rule:** when state has both a code-defined
default and a persisted cache, fixes go in the code default and
the cache key is bumped to invalidate stale copies. Editing the
cache fixes one machine and lies to you about the rest. **Same
trap for a config default:** flipping a default in code does
nothing on any machine that already has a persisted
`config.json` holding the old value — the persisted file shadows
the code default. Flipping a default means *also* migrating
existing persisted configs (or versioning the config so a stale
one is re-defaulted), not just editing the constant.

---

## 27. Consumer builds against the maintainer's source tree

**Symptom.** "I edited the library and nothing changed in the app
that uses it." Or the reverse: a consumer breaks the moment the
maintainer touches an unrelated file in the library's source
checkout. Or the source repo's diff is full of `*.o` / `dist/` /
`build/` churn.

**Looks like.** A stale build cache. An import-path bug. A
flaky linker.

**Actually is.** The boundary between the library's **source
tree** and its **deployed artifact** was never drawn. The
consumer is importing from / building against the maintainer's
live working checkout (or its `build/` directory) instead of a
published, version-pinned artifact. So the consumer floats on
uncommitted work-in-progress: it sees half-built state, breaks on
unrelated edits, and never picks up an intended change until the
build tree happens to be in the right state.

**Fix.** Separate the two and put a publish step between them:

1. Gitignore generated output (`build/`, `dist/`, `target/`,
   `*.o`, wheels, `__pycache__`). Source tree holds inputs, not
   outputs.
2. Publish releases explicitly: `tag → package → publish to the
   prefix / registry`.
3. Consumers depend on the **published** form by version/tag and
   bump deliberately — never a path into the maintainer's
   checkout, never another project's `build/`.

See [§13](delivery-rules.md#source-vs-deployed).

**Generalisable rule:** a consumer depends on a *published
artifact at a pinned version*, never on a producer's live build
tree. The publish step is the boundary that makes "edited but
unchanged" and "broke on an unrelated edit" both impossible.

---

## 28. Silently dropped function in a migration

**Symptom.** Months after a framework swap / language port /
service rewrite, something quietly doesn't work — a menu action
does nothing, an edge case isn't handled, a CLI flag is gone. The
feature existed in the old code; it never made it into the new
code. Nobody noticed because the migration "ran fine."

**Looks like.** A new bug. A regression from a recent change.

**Actually is.** The function was never ported. The migration was
declared done when the target *ran*, not when every source unit
was *accounted for*. And it's invisible now because the target has
since grown new functions — you can't eyeball original-vs-added
anymore, so the gap hides in the drift.

**Fix.** Inventory both sides — against the **source at its
pre-migration state**, not today's drifted target:

```sh
git -C <source> grep -hoE '^\s*(def|class|fn|pub fn) \w+' \
    <migration-base> -- <paths> | sort -u > /tmp/src.txt
grep -rhoE '^\s*(def|class|fn|pub fn) \w+' <target>/ \
    | sort -u > /tmp/tgt.txt
comm -23 /tmp/src.txt /tmp/tgt.txt    # in source, not in target → triage each
```

Triage every line: ported-renamed (add to the map), consciously
dropped (record why), or regression (port it). New target
functions don't appear in this list — additions are fine; the
only question is whether every *source* unit is accounted for.
See [§18.2](delivery-rules.md#migration-completeness).

**Generalisable rule:** "the target runs" proves nothing about
completeness. A migration is done when a source-vs-target surface
diff is empty modulo an explicit dropped-list — and you run that
diff *before* the target drifts, because every added function
makes the audit harder.

---

## 29. "Still broken" on a rig that never got the new build

**Symptom.** You fix a bug, deploy across the fleet, and a rig
still reports the old behaviour. You re-read the code, can't see
how it's still broken, burn an hour — then discover that rig was
running an old build the whole time. Or: a rig runs code that
exists on no commit anywhere, because it was deployed from an
uncommitted working tree.

**Looks like.** The fix didn't work. A new bug. A
rig-specific hardware quirk.

**Actually is.** **Version drift across targets.** Each rig is
deployed to independently (rsync / flash / bump), so at any moment
they run a *mix* of versions. Without a build stamp the running
system reports, "which version is this rig on?" is unanswerable —
so every cross-rig discrepancy starts as ambiguity between "real
bug" and "deploy gap," and you debug the code instead of checking
the deploy.

**Fix.** Stamp the build and make the target report it:

1. Write a stamp at build/deploy time (`git_sha`, `git_dirty`,
   `built_at`, `built_by`) into the artifact. `git_dirty: true`
   is the field that catches deploy-from-uncommitted-tree.
2. The running system exposes it — `--version`, a startup log
   line, a status field, `GET /version`.
3. On any "works on A, not B," **diff the two rigs' stamps
   first.** Same sha → real bug; different sha → deploy gap, stop
   reading code and finish the deploy.

See [§12.1](delivery-rules.md#build-version-stamp).

**Generalisable rule:** anything deployed to more than one place
reports a build stamp at runtime. "Is this the right version" must
be a command you run, never a belief you hold. Silent version
drift is the deploy-side twin of the dependency-lock drift
(#22) and the layout-cache drift (#26): authoritative state and
deployed state diverge unless something makes the gap loud.

---

## 30. Behaviour / code path / device selected by an env var, drifting across rigs

**Symptom.** A feature works on one rig and not another with the
same code. Or a behaviour silently changes after a redeploy. Or
nobody can answer "what is this rig actually configured to do?"
The canonical instance: an engage path that works on rig A but
hangs on rig B because a topic-forwarding list (`BRIDGE_OUT` or
similar) was set in rig A's environment and never in rig B's.

**Looks like.** A rig-specific hardware quirk. A real bug present
on only one rig. A flaky deploy.

**Actually is.** **Per-deployment behaviour lives in an
environment variable.** Each rig's env is set out-of-band (shell
profile, systemd unit, launch script), so the rigs drift with no
record anywhere. The config isn't committed, isn't versioned,
isn't greppable, isn't reviewable — and an unset var silently
falls back to a default, so a rig that never had it set behaves
differently with no signal.

**Fix.** Get the choice out of the environment, in this order:
- **Derive it in code from hardware/role** when it tracks the
  machine — the `device.py` pattern (`preferred_device()` →
  CUDA/CPU from the real target; `USE_TINYGRAD_WARP = JETSON and
  not TICI`). "Which code runs" becomes a computed property of
  *what this machine is*, identical on every rig of that class,
  nothing to set or forget.
- **Else a committed, versioned config file** read via a typed
  loader (service TOML, Params store, `<rig>.config.json`). "Why
  do these two rigs differ?" becomes a file diff, reviewable in a
  PR, travelling with the code.

Env stays only for one-shot debug overrides and OS-owned
location/secrets — never the source of truth for behaviour or a
code path. See [§7.5](engineering-rules.md#env-var-config).

**Generalisable rule:** **default to no new env var** — if flipping
a value changes *which code runs*, it belongs in code keyed on
hardware/role (derived, not set) or a checked-in config file, never
the environment. Reaching for a behavioural env var means the wrong
design; the only OK env is a removed-before-merge debug nudge or a
pure location/secret. Test: "if two rigs differ, can I see why by
reading committed files?"

---

## 31. Lost track of which services exist / are running

**Symptom.** Nobody can answer "what services should be running on
this rig?" A daemon died hours ago and no one noticed. An orphan
from a crashed parent is still alive, posting duplicate offers or
holding a stale resource. A service is started in a launch script
*and* a systemd unit *and* by hand — and they disagree.

**Looks like.** A flaky daemon. A resource conflict. A
rig-specific gremlin.

**Actually is.** **No single source of truth for the service
set, and no reconciliation against reality.** Services accreted
across launch scripts, units, parent-spawns, and hand-starts, so
"the full set" is a union nobody can see, and "what's running"
has drifted from "what should run" with nothing to catch it.

**Fix.**
1. **One declarative table** lists every service with its gate and
   rationale; the supervisor reads it; nothing starts a service
   behind its back. Not in the table → shouldn't be running.
2. **A reconciliation probe** prints the three-way diff:
   declared-but-not-running (crashed / gated-off), running-but-not
   -declared (orphan / ad-hoc), declared-and-running (ok). "Is
   everything up that should be, and nothing else?" becomes one
   command.
3. **Orphan-sweep at startup** ([§8.11](ui-rules.md#orphan-sweep))
   is the enforcement half; the probe is the read half.

See [§6.1](engineering-rules.md#service-inventory).

**Generalisable rule:** the service set is declared in one
versioned place, and "running vs. declared" is a command, not a
memory. When they diverge, the table wins — reconcile reality to
it, or change the table on purpose.

---

## 32. "The link is up" but no data is flowing

**Symptom.** A sensor / peer / feed reads as dead, yet every
link-level check says the transport is fine — the bus is up, the
socket connected, zero errors. So the transport gets ruled out
and the search goes everywhere else.

**Looks like.** A decoding bug downstream. A subscriber that isn't
subscribing. A config mismatch on the consumer.

**Actually is.** **Link healthy, application silent.** The wire is
up and error-free, but the *producers* on the far end aren't
transmitting — no power/wiring to the devices, a missing
enable/wakeup TX they need before they stream, or they're simply
not running. A link-layer check (`ERROR-ACTIVE, 0 bus-errors`)
answers a different question than "are frames arriving" (`0
frames`), and only the second one matches the symptom.

**Fix.** Check **both layers, separately**:

```sh
ip -details -statistics link show <iface>   # layer 1: UP? errors?
timeout 5 candump <iface> | head            # layer 2: frames flowing?
```

`link=UP frames=0` localises the fault to the producer side in
one line. A liveness probe must assert on **frames received**, not
link state. And don't trust placeholder IDs/addresses as known
until a real on-target scan shows the live ones.

See [§15.5](delivery-rules.md#link-vs-data).

**Generalisable rule:** "up" and "carrying data" are different
questions on every transport. A health check that only proves the
link is configured will call a dead feed healthy. Assert on data
crossing. (Transport-side twin of #17, daemon-runs-but-publishes
-nothing.)

---

## 33. Aggregate health metric hides a single dead element

**Symptom.** One of N parallel elements is dead — a single camera
tile stale for minutes while the others are fresh, one consumer
wedged while the queue still drains, one rig offline in a fleet
that reports "healthy." No watchdog fired; the system thought it
was fine.

**Looks like.** A flaky element. A transient that'll recover. A
monitoring blind spot specific to that element.

**Actually is.** Liveness is measured **in aggregate**, and the
healthy elements keep the aggregate moving so it never trips. A
global "total frames flowing" stays green while one stream is
dark; a "queue is draining" stays green while one consumer is
stuck; a "all rigs reported in the last hour" stays green while
one rig has been dead for 59 minutes. The sum is exactly what
masks the single failure.

**Fix.** Track liveness **per element**, not as a sum:

```python
# WRONG — one global counter; one dead element is invisible
if total_frames_advanced(): ok()

# RIGHT — per-element last-progress; a single stale one is caught
for el in elements:
    if now() - el.last_progress_ns > STALL_TIMEOUT:
        escalate(el)        # capped per-element recovery, §10.2
```

The global metric is fine as a *coarse* "is anything alive at
all"; it is never sufficient for "is *everything* alive." See
[§10.2](engineering-rules.md#capped-recovery).

**Generalisable rule:** any health check over N things must be
per-thing. An aggregate over the set answers a different question
than the one the operator is asking ("is *this* one working"), and
the healthy members will always hide the dead one. Same family as
"link up ≠ data flowing" (#32) and daemon-runs-but-publishes
-nothing (#17): assert on the specific thing, not a proxy that
stays green.

---

## 34. A check that's always red, so everyone ignores it

**Symptom.** A CI workflow has been failing for weeks; the badge
is permanently red; PRs merge anyway because "that one's always
broken." Then a real regression turns it red *for a reason* — and
nobody notices, because red was already the normal state.

**Looks like.** A flaky CI environment. A known-broken test
someone will get to eventually. Harmless background noise.

**Actually is.** A **dead signal occupying a live signal's slot.**
A status that's always asserting (red CI, an alert that fires
every boot, a "degraded" badge never seen green) carries zero
information — worse than absent, because it sits where a working
check would be and has trained everyone to scroll past it.

**Fix.** Make it green-in-the-good-state or delete it:

- A workflow that can't pass on the current codebase is **removed
  until someone owns making it pass** — not left red "to fix
  later." (We deleted several perpetually-failing CI workflows +
  a dead Jenkins job across repos for exactly this.)
- A new check **lands green** — never add one that's red on day
  one.
- The sentence "oh, that's always failing, ignore it" is the bug
  out loud. The day a team says it, the check has negative value:
  fix or remove it that day.

See [§16.6](delivery-rules.md#red-is-no-signal).

**Generalisable rule:** a signal only has value if its alerting
state is rare and actionable. A permanently-red check is noise
wearing a signal's costume — green-in-good-state or gone. (Macro
form of #19, per-tick log spam, and "don't ERROR for a recovered
condition": a thing that's always asserting tells you nothing.)

---

## 35. Translated / non-Latin text renders as boxes, or all text looks thin & gray

**Symptom.** Two faces of the same font gap:
- After enabling another language, the translated UI shows boxes
  ("tofu") where the text should be — or stays English even
  though the catalog has the strings.
- All text renders faint / washed-out gray at normal sizes,
  chased as a theme-contrast bug.

**Looks like.** A broken translation catalog; a theme/contrast
problem; a rasteriser bug.

**Actually is.** A font problem, two variants:
- **Tofu:** the bundled font doesn't cover the script (CJK, etc.)
  — every glyph outside its coverage is the missing-glyph box.
  The catalog is fine; the font can't draw it.
- **Thin & gray:** the bundled asset is a **variable font** whose
  `wght` axis defaults to Thin (100), and the rasteriser
  (`stb_truetype`, no FreeType) can't apply the axis — so it
  renders weight-100 everywhere, which on a wide CJK face looks
  gray.

**Fix.**
- Tofu → bundle one Unicode-wide font covering every script you
  ship a catalog for; adding a language is catalog **+**
  glyph-coverage check ([§8.14](ui-rules.md#fonts),
  [§8.15](ui-rules.md#i18n)).
- Thin/gray → bundle a **static instance** at the weight you want,
  not the variable file:
  `fonttools varLib.instancer VF.ttf wght=500 -o Medium.ttf`. A
  build-time check that the bundled asset is the instanced file
  (size/name) prevents the regression.

**Generalisable rule:** text rendering has three independent
prerequisites — the **string** (catalog), the **glyph** (font
coverage), and the **weight/axis** (static instance the rasteriser
can draw). A failure in any one looks like a failure in the
others; check all three. "Looks fine on my machine" = dev locale +
dev theme + dev font; production is every locale, theme, and
script you ship.

---

## 36. Wrong-platform shared library linked into a build

**Symptom.** A bundle built / run on one platform fails to load a
native lib — `wrong ELF class`, `mach-o … but wrong
architecture`, `image not found`, `incompatible architecture`,
`%1 is not a valid Win32 application` — or it loads and crashes
deep in the first FFI call. Works on the machine it was built on,
breaks on every other.

**Looks like.** A corrupt download. A missing dependency. A
broken install on the user's side.

**Actually is.** A `.so` / `.dylib` / `.dll` from the **wrong
platform or arch** got linked into the bundle: a macOS build
carrying a Linux `.so`, an arm64 bundle with an x86_64 lib, a lib
built against a libc/runtime version the target doesn't have.
Native artifacts aren't interchangeable; one slipped in because
the build globbed a shared `lib/` that the last (wrong-platform)
build had populated, or a built lib was committed into source and
a cross-platform build grabbed it.

**Fix.**
- Lay libs out **per platform+arch** (`macos-arm64/`,
  `linux-x86_64/`, …) and have the packager select by `(os,
  arch)` — never glob one shared directory.
- Don't commit a built `.so`/`.dylib`/`.dll` into source as "the"
  lib (§13 source-vs-deployed: generated, gitignored, selected at
  package time).
- Build the lib on/for the platform that runs it (§11).
- **Verify the linked arch** in the build/selftest (`file`,
  `lipo -archs`, `otool -L`, `readelf -h`, `ldd`) so a mismatch
  fails the build, not the operator's launch.

See [§11.1](delivery-rules.md#platform-libs).

**Generalisable rule:** a binary artifact is valid only for the
exact `(os, arch, libc/runtime)` it was built for. A cross-platform
product ships one lib set per platform and picks by platform at
package time; mixing fails — often silently, always on someone
else's machine. (Build-time twin of the §12/#4 rsync clobber: same
wrong-platform-artifact root, caught at packaging instead of at
deploy.)

---

## 37. Authless "LAN-only" service the edge can actually reach

**Symptom.** An internal service with no authentication — a
package registry, admin port, metrics endpoint, debug API —
turns out to be reachable from outside the trusted network. Often
discovered by a scan, an audit, or an incident, not before.

**Looks like.** A safe internal tool ("it's only on the LAN").
A firewall's job, handled elsewhere.

**Actually is.** The "the subnet is the auth boundary" assumption
was never enforced where it matters. The service binds `0.0.0.0`
instead of the LAN IP; or the cloud edge *doesn't forward* it
today but is one config edit from doing so; or a VPN/relay quietly
bridges the trusted subnet to somewhere it shouldn't. Authless is
fine **only** while the gate is real — and here it wasn't.

**Fix.**
- Bind the **LAN interface**, never `0.0.0.0`.
- Make the internet-fronted edge **actively reject** the private
  surface (e.g. the relay refuses the reserved `_pkg/` key
  prefix), so a stray proxy/forward still can't leak it — a code
  block, not just a firewall rule someone can widen.
- Treat any topology change (new forward, VPN bridge, edge
  reconfig) as **invalidating** the trusted-network assumption
  until re-checked; if the service can be reached off-subnet, it
  needs real auth.

See [§23.1](delivery-rules.md#subnet-as-auth).

**Generalisable rule:** subnet-gated authless is a property of the
*boundary*, not the service. It holds only while the edge
provably can't route in, enforced at the edge in code — not
assumed at the bind. The day the edge can reach in, authless is an
open door.

---

## 38. GPU hang masquerading as "the whole pipeline is dead"

**Symptom.** Everything downstream of a model / accelerator stage
goes silent at once — no inference output, and the consumers of
*its* output look dead too. The process is **alive** with
multi-hour uptime; the supervisor never restarted it. You start
debugging the messaging layer, the consumers, the wire.

**Looks like.** A dead message bus, a broken subscriber, a
network fault — because everything downstream is starved at once.

**Actually is.** The accelerator wedged. A GPU/NPU driver fence
never signalled (often under memory pressure), so the inference
process is **blocked, not crashed** — a clean crash would have
been auto-respawned; a fence-wait hang is invisible to a
process-liveness check. Nothing downstream is broken; it's all
starved of the one input that stopped.

**Fix.** Diagnose at the driver, not the app:

```sh
cat /proc/$(pgrep -f <inference-proc>)/wchan
# "dma_fence_default_wait" → blocked on a GPU fence. Restart the
# service; stop debugging downstream.
```

Then prevent the recurrence: keep accelerator memory headroom so
the driver returns errors instead of wedging; don't run two
accelerator contexts contending in one process; put a heartbeat
on the model's **output topic** so the hang is caught by
output-liveness, not by a PID check that always passes.

See [§9.5](engineering-rules.md#accelerators).

**Generalisable rule:** a GPU/NPU hang is "alive but blocked,"
which a process supervisor reads as healthy — the accelerator
instance of "looks healthy ≠ working." Detect it on output
produced (and at `wchan`), never on the process being up; when a
whole downstream fan-out dies together, suspect the shared input
(the accelerator), not each consumer.

---

## 39. Build-time backend env var (`CUDA=1`) bakes the wrong artifact

**Symptom.** A rig you believe is GPU-accelerated runs CPU-slow
(or a CPU rig got a GPU build that won't load). Two build hosts
produce different binaries from the same source. Or flipping the
flag and rebuilding gives you the *other* backend's stale
objects. The finished binary looks identical either way.

**Looks like.** A runtime config problem, a slow GPU, a flaky
rig — anything but the build.

**Actually is.** A bare env-var backend switch (`CUDA=1 scons`,
`USE_GPU=1 make`) read straight from `os.environ`. It forks the
*artifact* at build time but is invisible afterward, undeclared
to the build system, and silently CPU-fallback when unset. Three
ways it bites: drifts between build hosts (each shell differs),
falls back silently (unset ≠ error), and the build cache doesn't
rebuild on the flip (a bare env read isn't in the dependency
graph).

**Fix.**
- Make it a **declared build variable** the build system records
  and cache-keys on (`scons gpu=1` via `Variables()`,
  `-DUSE_CUDA=ON`, a Cargo feature) — not `os.environ["CUDA"]`.
- **Default explicitly and log the chosen backend at build time**
  (`>>> building CUDA=ON` / `CPU fallback — no toolkit`).
- **Stamp the backend into the artifact** (`build_version.json:
  "accel"`) so "is this the GPU build?" is `--version`, not a
  guess (§12.1).
- Keep **per-backend build dirs**; don't link GPU objects into a
  CPU build from one shared `lib/` (§11.1).

See [§9.6](engineering-rules.md#build-backend-switch).

**Generalisable rule:** a flag that *forks the build* is a build
input, not a runtime toggle — declare it, cache-key it, log its
default, stamp it into the output. An undeclared `os.environ`
read that selects a backend drifts across hosts and falls back in
silence. (Build-time meeting of the env-var-config (#30),
silent-falsy (#3) and authoritative-vs-actual (#1) cycles.)

---

## 40. "GPU" work silently running on the CPU (and "loaded" hiding 0 Hz)

**Symptom.** A pipeline you believe is GPU-accelerated is slow,
or one core is pegged — but the model log says `"models loaded in
1.0 s"` and nothing errors. Switching to the GPU device "to fix
it" makes the model stall to **0 Hz** instead. The messages never
say which device actually ran.

**Looks like.** A perf-tuning problem, a slow GPU, a model too
big — anything but "it's on the CPU."

**Actually is.** Two device-selection lies:
- An **auto/DEFAULT device selector** (`CL_DEVICE_TYPE_DEFAULT`)
  resolved to the **CPU** device on a runtime exposing both, so
  the "GPU" kernels run on CPU at many times the cost — silently.
- The init message **`"loaded"` is not a liveness signal.** A
  model that loads then publishes 0 Hz (two accelerator contexts
  contending per frame, §9.5 rule 1) looks identical at load to a
  healthy one — so "I moved it to the GPU and it loaded fine"
  hides that it produces nothing.

**Fix.**
- **Select the compute device explicitly** (`CL_DEVICE_TYPE_GPU`
  / a named CUDA device), never DEFAULT/auto.
- **Log which device bound, by name**, at startup — and announce
  the active compute path (`"tinygrad-CUDA preprocessing active —
  OpenCL bypassed"` / `"CPU fallback: no GPU device"`).
- **Gate on output rate, not the load message** — heartbeat the
  model's output topic (§16.4). 0 Hz with "loaded" printed = a
  contention stall, not a successful move.
- If GPU preprocess + GPU inference contend, **share one context**
  (same stream) or keep preprocess on CPU — don't run two
  accelerator users per frame (§9.5 rule 1).

See [§9.5](engineering-rules.md#accelerators) (rule 6).

**Generalisable rule:** which device your compute ran on is
**silent unless you make it loud** — select explicitly, log the
bound device, verify on output rate not on "loaded." A device
change you can't confirm from a runtime message is unverified.
(Accelerator face of "looks healthy ≠ working," cycle #2.)

---

## 41. Compatibility gated on a peer's version string, not its features

**Symptom.** A consumer warns `peer reports 3.2.0, expected 3.1.1`
against a peer that is **newer and working fine**. Every release,
someone has to hand-sync a version constant in one repo to match
another. A perfectly good upgrade trips a false "incompatible."

**Looks like.** A real version mismatch, a botched deploy, a
config error.

**Actually is.** The consumer hard-mirrors the producer's version
in a constant (`EXPECTED_PEER_VERSION`) and gates on it
(`!=` / `<` / `==`). The two live in **different repos**, so the
constant drifts the instant one bumps without the other — and a
`<`/`!=` compare treats a *newer* (forward-compatible) peer as a
fault. Doubly wrong: drifts by construction, and rejects valid
upgrades.

**Fix.** Negotiate on **capabilities**, not version:
- Producer advertises `{service, version, features:[…]}`.
- Consumer declares a **required feature set**; compatible iff
  `required ⊆ advertised`. Depend on a new endpoint → add its
  *name* to the set. That set is the contract.
- The version string is **cosmetic** — banners/logs only, never a
  gate. Verify the `service` field too (so a probe hitting the
  wrong port fails loudly).
- **One verdict function**, called everywhere; a second ad-hoc
  `version != EXPECTED` check elsewhere reintroduces the drift.

See [§12.2](delivery-rules.md#version-contract).

**Generalisable rule:** cross-component compatibility is
`required features ⊆ advertised`, decided in one place; the
version is a label, not a gate. A mirrored peer-version constant
is the authoritative-vs-actual cycle (#1) across a repo boundary —
it drifts the moment the two repos release independently.

---

## 42. The CPU↔GPU (or any backend) oscillation

**Symptom.** A stage keeps flip-flopping across commits: moved to
the GPU, reverted as "too slow / 0 Hz," moved again, reverted
again. Days go into a decision that never settles, and each round
starts from scratch.

**Looks like.** A genuinely hard perf tradeoff that just needs one
more attempt.

**Actually is.** The decision was never *measured* (the wall-clock
number that triggered a flip wasn't attributed — "220 ms" was a
framework re-tracing its graph in Python every frame, not GPU
time; cached it was ~14 ms), the switch is a *rebuild* (so each
experiment costs a deploy cycle, and the result isn't even
trusted), and no prior round was *recorded* (so the next person
re-derives it). Three compounding gaps, not a hard problem.

**Fix.**
- **Measure with attribution** — kernel vs host-launch vs
  per-frame graph build vs context thrash — before switching. A
  flip on an un-attributed number is a coin toss.
- **Default to the *validated* path; keep the other a runtime
  switch, not a rebuild.** On hardware with the accelerator the GPU
  path is the default *once validated on the target*; CPU is
  bring-up scaffold or a **loud, alarmed degraded mode**, never the
  silent steady state, and a validated decision re-opens only on new
  attributed measurement (§9.10). A config toggle makes an
  experiment one restart, not rebuild-redeploy-revert.
- **Record the decision, its numbers, and the blocker** that would
  change it (e.g. "GPU = 0 Hz: two CUDA contexts contend, needs
  one stream"). Then it's answered by reading, not re-running.
- **Confirm the stage is the bottleneck at all** — porting a
  non-saturated stage to the GPU is motion, not progress.

See [§9.7](engineering-rules.md#cpu-gpu-decision).

**Generalisable rule:** a recurring "move it / revert it" is the
signature of an *unmeasured, unrecorded, rebuild-gated* decision.
Measure with attribution once, default to the **validated** path
(the accelerated one on hardware that has the accelerator — §9.10),
make the alternative a logged runtime toggle with a loud degraded
fallback, and write down the verdict + its blocker — the
oscillation is a process failure, not a perf mystery.
(Decision-oscillation cycle #6; the general rule is §18.3.)

---

## 43. Build-tagged device ≠ runtime device → JIT-mismatch crash-loop

**Symptom.** A GPU stage (a JIT-compiled model, a kernel cache)
crash-loops on the target: it loads, then dies mid-init, restarts,
dies again. No output → everything downstream goes NaN/empty. The
binary built "fine."

**Looks like.** A bad model export, a driver bug, an OOM, a flaky
target.

**Actually is.** The **build** tagged the artifact for one device
and the **runtime** expects another. Two places resolved the
device independently and drifted — e.g. the build's arch-map
defaulted to CPU while the loader hardcoded `DEV=CUDA`, producing
a CPU-tagged blob that JIT-mismatches the CUDA runtime. The
artifact is internally inconsistent with the process that loads
it.

**Fix.** One device resolver that **both** build and runtime
import, so they agree by construction:
- the build forces that device on every path (prebuilt-guard
  **refuses** a wrong-device artifact; compile path **asserts**
  it);
- the artifact carries its device in a sidecar the runtime reads
  back → a mismatch is a **loud refusal at load**, not a
  crash-loop;
- record the per-process GPU/CPU allocation as policy next to the
  process table; a CPU fallback for a GPU-designated stage is a
  regression to fix, not a knob.

See [§9.8](engineering-rules.md#device-single-source).

**Generalisable rule:** when a build and a runtime each pick a
device/target/ABI independently, they drift and the artifact
mismatches its loader — resolve it in one shared place, fail loud
on mismatch at load. (Authoritative-vs-actual cycle #1, between
the build and the runtime.)

---

## 44. Widened a load-bearing throughput cap to use "idle" capacity → starved the latency-critical peer

**Symptom.** A process is pegging one core while others sit idle
(or a rate/batch cap is "leaving performance on the table"). You
lift the cap — give it more cores, raise the rate. Its own
throughput goes up; then a *different*, safety- or latency-critical
process collapses (rate halves, goes 0 Hz), load spikes, and the
board may watchdog-reboot.

**Looks like.** An obvious inefficiency someone forgot to fix —
free performance just sitting there.

**Actually is.** The cap was **load-bearing**: it throttled that
process's draw on a *shared, invisible* resource (memory bandwidth
on a unified-memory SoC, a GPU/EMC budget, an I/O bus) to leave
headroom for the critical peer. The two don't contend for CPU
cores — they contend for the bus — so "five cores are idle" was
never the relevant budget. `nice`/affinity can't fix it afterward
either: CPU priority doesn't arbitrate memory bandwidth. (We also
hit the build form: renicing a heavy build to 19 still starved the
model — "separate build dir" ≠ "won't interfere," same bus.)

**Fix.**
- Before lifting an apparent inefficiency, **ask what consumer it
  protects** — a deliberate constraint usually is one.
- **Measure the change against the *protected* consumer's output
  rate**, not the throughput of the thing you sped up. A gain that
  regresses the protected peer is a regression; revert it.
- To win back the shared resource, **reduce the contender's draw**
  (cap throughput, move fewer bytes, rate-limit before the
  expensive stage) or **don't co-locate** heavy work with the
  latency-critical process (stop the service, do the heavy work in
  a maintenance window). Never reach for `nice`.
- **Annotate the constraint in place** — what it protects, what
  breaks if widened — so the next reader doesn't "optimise" it away.

See [§9.9](engineering-rules.md#shared-resource-contention).

**Generalisable rule:** on a shared-memory board, throughput trades
against a latency-critical peer through a resource the obvious
metric (idle cores) doesn't show — a constraint that throttles one
process may be protecting another. Find what it protects and watch
the protected consumer when you change it; CPU priority won't save
you from bus contention.

---

## 45. Service stayed dead after a clean-exit race, or restart-looped a fault only a reboot clears

**Symptom.** Two ends of the same wrong restart policy. (a) After a
reboot the stack silently never came up — the unit sits
`inactive(dead)`, no crash in the log, a manual `systemctl start`
fixes it. (b) Or the opposite: a daemon restarts over and over
against a hardware that's wedged, never recovering, burying the
real cause under restart noise.

**Looks like.** (a) A failed boot / bad install. (b) A flaky daemon
that just needs one more restart.

**Actually is.** (a) The unit is `Restart=on-failure`, but the
process **exited 0** — a boot-race (started before a dep was ready)
or an exit-to-restart pattern. `on-failure` does not fire on a clean
exit, so nothing restarts it. (b) The fault is **not
software-recoverable** — a hung sensor firmware, a wedged USB
device, a fence-blocked GPU. A process/stack restart, even a USB
re-enumerate, doesn't power-cycle the hardware; only a reboot (or a
physical replug) clears it, so software restarts spin forever.

**Fix.**
- For any daemon whose normal/transient exits can be **code 0**, use
  **`Restart=always`** — a deliberate `stop` still stops it, and a
  reboot self-recovers. If the unit is generated by a bootstrap
  script, fix the **template**, not the live `/etc` copy.
- Bound software retries by **recoverability**: retry the
  transient-resource case (§6 loop); for a fault still dead after N
  restarts, stop looping and **surface it loudly** for escalation
  ("sensor firmware hung — reboot required") on the journal and any
  operator surface.

See [§6.2](engineering-rules.md#restart-policy).

**Generalisable rule:** match the restart policy to how the process
actually exits (`always` when a clean exit can be transient, so a
reboot self-heals), and don't software-restart a fault only a
power-cycle can clear — detect it and escalate, never loop in
silence.

---

## 46. "Hard floor" / "can't optimize" that was the wrong tool or a misattributed cost

**Symptom.** Work stops on a perf problem because someone concluded
it's fundamental: "that's a hardware floor," "this stage can't go
faster," "lowering the rate won't help." The verdict sticks and
nobody revisits it — until a one-line change (a different pipeline
element, a cached/compiled path) blows past the "floor."

**Looks like.** A settled engineering fact about the hardware.

**Actually is.** An *unattributed* limit. The cost was an artifact
of the **wrong tool** (a copy-through element where a zero-copy one
exists), a **misattributed number** (a framework re-tracing its
graph counted as device time — §9.7 r1), or a real sub-floor for
**one lever** over-generalised to the whole. Often it hides a
*superset assumption* — "the general engine does everything the
fixed-function unit does" — that's simply untrue.

**Fix.**
- A floor claim must name **the mechanism** (physics / the
  fixed-function unit / a per-item dequeue copy — not "it's slow")
  **and the alternatives ruled out** (other element, other backend,
  zero-copy route). No mechanism + no ruled-out list = a guess that
  freezes everyone.
- Attribute the cost before concluding (§9.7 r1), then try the one
  alternative path you haven't.

See [§15.6](delivery-rules.md#optimization-floor).

**Generalisable rule:** "it can't go faster" stops everyone from
looking, so it must cite its mechanism and what it ruled out — an
unattributed floor is usually the wrong tool or a misattributed
cost, more expensive than the slow path it was defending.

---

## 47. Rebooted to make it go away — lost the evidence, the bug came back

**Symptom.** Something misbehaves; the reflex is "reboot the rig" /
"restart the service" / "power-cycle it." The symptom clears, work
resumes — and the same fault returns later, with no more
understanding of it than the first time.

**Looks like.** A fix. "Turned it off and on again — all good now."

**Actually is.** A recovery that **destroyed the evidence**. The
reboot wiped the kernel ring, the journal, the stuck `wchan`,
`/proc` state, the wedged device's registers — exactly what would
have localised the cause — and converted a reproducible bug into an
intermittent ghost. "Symptom gone" got mistaken for "fixed"; the
bug is still open and will resurface.

**Fix.**
- **Capture before you reset** — dump journal / `dmesg`,
  `/proc/<pid>/wchan` + status, device and process state to a file
  that survives the reboot.
- **Escalate narrowest-first** — re-open the handle / reset the
  device / restart the one daemon / **re-init the wedged subsystem's
  driver (re-bind, re-train)** before a full reboot; and note a
  *warm* reboot may not power-cycle a co-processor/SerDes, so the
  driver re-init is often narrower *and* more effective than a reboot
  (§6.2, §15.7).
- When you do reboot, **record what justified it** so it's a data
  point, not a habit.

See [§15.7](delivery-rules.md#no-reflexive-reboot).

**Generalisable rule:** a reboot/restart that makes a symptom vanish
without a named cause has destroyed the evidence and closed nothing
— capture volatile state first, recover with the narrowest action,
and reboot deliberately as a last resort. (Diagnostic-first cycle,
§15.1, applied to recovery actions, not just code edits.)

---

## 48. Consumer alive but publishes nothing — correlated streams stamped from different clocks

**Symptom.** A consumer that joins several streams (a model syncing
a road + wide camera, a fuser, a log merge) is alive and busy —
pegging CPU — but never emits output. Logs spam "frames out of
sync." Downstream goes NaN/empty because the consumer's topic is
silent.

**Looks like.** The consumer is broken — a parser bug, a deadlock,
too slow.

**Actually is.** The inputs are stamped from **different clocks /
epochs**, so the consumer's timestamp match never succeeds. Each
producer used its own zero (a per-pipeline `base_time`, a
per-process start), so frames that arrived together carry
timestamps seconds apart; with a tight match tolerance the gap
never closes and the consumer spins forever. (Or a unit slip — µs
where ns expected — desyncs them the same way.)

**Fix.**
- Stamp every correlated stream from **one shared monotonic clock**
  with one epoch (`CLOCK_BOOTTIME` / `nanos_since_boot()`) at the
  common boundary — not a per-pipeline/per-source timestamp.
- Keep the unit ns end-to-end and name it (`*_ns`).
- When a *correlating* consumer stalls with no output, suspect the
  **timestamp contract upstream** before the consumer.

See [§2.1](engineering-rules.md#timestamp-contract).

**Generalisable rule:** timestamps that are ever compared must share
one clock, one epoch, and one unit; a per-source clock silently
desyncs correlated streams and stalls the consumer that joins them —
looking exactly like the consumer's own bug. (Looks-healthy ≠
working cycle #2.)

---

## 49. A subsystem fault blamed on that subsystem — the real cause was resource starvation elsewhere

**Symptom.** A box reboots (or a subsystem wedges) and the crash log
names a subsystem: "GPU PMU timeout," "watchdog." The newest code in
that subsystem is the obvious suspect, so it gets reverted — and the
fault continues, or the revert "fixes" it by accident.

**Looks like.** A bug in the subsystem the fault names — usually
whatever changed there most recently.

**Actually is.** That subsystem was a **victim of starvation
elsewhere.** In the case that taught this: a `printk` kthread
draining ~150/s `WARNING` to a 115200-baud serial console pinned a
CPU core at 97 %, which starved the GPU's CPU-side PMU servicing →
PMU timeout → the driver software-rebooted the SoC. The GPU code was
a red herring; the cause was a **log line on a slow sink.**

**Fix.**
- **Capture the evidence that survives the reboot** before touching
  code — `/sys/fs/pstore/console-ramoops-*`, `dmesg`, load/per-core
  CPU (§15.7) — and **attribute the mechanism** (§15.6) before
  reverting the loudest/newest suspect (blind-revert is the
  decision-oscillation shape #6).
- Keep high-rate logs off slow sinks (low `console_loglevel`,
  rate-limit the source) so logging can't pin a core (§16.7).
- Treat cross-subsystem **resource starvation** (a shared bus, a
  shared core, memory bandwidth) as a first-class suspect for a
  fault that names an innocent subsystem (§9.9).

See [§16.7](delivery-rules.md#slow-log-sink).

**Generalisable rule:** a fault that names a subsystem is not proof
the bug is in it — resource starvation cascades, so attribute the
mechanism from captured evidence before reverting the obvious
suspect; the loudest/newest component is often the victim, not the
cause.

---

## 50. Frozen app missing a whole subsystem — a native dep was absent at bundle time

**Symptom.** A packaged / frozen build runs fine from source and
"builds clean," but on a clean target it dies immediately with
`ImportError: cannot import <core module>` — often something
central (the transport layer, the config loader) you obviously
depend on.

**Looks like.** A missing Python package, a broken import path, a
code bug, a bad target.

**Actually is.** The freezer (PyInstaller etc.) discovers what to
bundle by **importing** your code. A compiled extension `dlopen`s a
**native shared library** that wasn't installed in the build
environment, so `import <that module>` raised at *analysis* time —
and the bundler **silently dropped** the module (and its dependents)
instead of failing. The build succeeded missing a subsystem; you
find out at first run on a machine that doesn't happen to have the
`.so`.

**Fix.**
- Install every **native** runtime dep the bundled extensions
  `dlopen` (the `.so` / `.dylib` / `.dll`) in the build image — so
  each is present at *collect* and *run* time — and document why
  each is needed.
- **Verify by importing the entry points in the frozen artifact**
  on a clean machine, not by the build exit code (§11.2, §18.1).
- Read the freezer's analysis warnings — a "could not import /
  module not found" line during the build is the silent drop
  announcing itself.

See [§11.2](delivery-rules.md#freeze-native-deps).

**Generalisable rule:** a freezer ships only what it can import, so
a missing native dependency silently deletes a whole module from the
artifact — the build passes, the app fails on a clean target.
(Wrong-form-artifact cycle #4 + looks-healthy ≠ working #2: the
build is green, the artifact incomplete.)

---

## 51. Closing the client stopped the always-on service

**Symptom.** After using a tool / GUI, the underlying service is
*down* — the next session finds the rig's stack stopped, or a
feature dead. Nobody ran an explicit stop; they just closed the app,
or a new launch superseded an old one.

**Looks like.** A crash, a flaky service, a bad boot.

**Actually is.** The client's **teardown path stopped the
service.** A cancel / app-exit hook / superseding-relaunch ran
`systemctl stop <service>` as cleanup. The unit was
`Restart=always`, but an *explicit* stop defeats that (the
supervisor honours an operator stop), so it stays down. Sibling
failure: the client never *reconnects* after the service
legitimately restarts (redeploy, onboard, reboot), so it goes stale
after one connect until an operator re-triggers.

**Fix.**
- A client's teardown **detaches** (drops the connection / log
  tail); it does not `stop` a supervisor-owned service. Operate on
  the service with **restart/reload**; reserve `stop` for a
  deliberate operator action. (§6.3.)
- A long-lived client **auto-reconnects / re-spawns its worker**
  across the service's restarts, rate-limited — it doesn't assume
  one connect lasts forever. (§6.3, §10.2.)

See [§6.3](engineering-rules.md#always-on-contract).

**Generalisable rule:** an always-on service's lifecycle belongs to
its supervisor — a client must never stop it from a teardown path
(an explicit stop defeats `Restart=always`) and must reconnect
across its restarts. (The lifecycle cousin of §6.2.)

---

## 52. Buffered playback on a control feed — the operator is acting on stale reality

**Symptom.** A teleop / takeover video looks smooth, but the
operator reports the vehicle "responds late," or a tile looks live
yet lags the world by a noticeable margin. Inputs feel disconnected
from what's on screen.

**Looks like.** Network latency you can't help. A slow decoder. A
laggy display.

**Actually is.** The receive/display path **buffers frames** — a
FIFO playback queue, a too-deep decode queue, a backlog-drop
threshold tuned for smoothness — so it trades latency for
smoothness. Fine for *watching*, dangerous for *controlling*: every
queued frame is reality the operator is behind, and on a control
loop that's a safety issue, not a cosmetic one.

**Fix.**
- Display from a **single-slot latest-wins** buffer; drop stale
  frames instead of playing them.
- Bound the receive backlog + decode queue to a small **time**
  budget (a small multiple of baseline latency); past it, drop to
  newest — accept **lower fps, not growing lag**.
- If the link can't keep up, lower the source **encode
  resolution/bitrate** (widest stream first); don't buffer.
- Mark the tile **stale** when the newest frame ages out, rather
  than showing a frozen "live" view.

See [§3.4](engineering-rules.md#control-feed-freshness).

**Generalisable rule:** a feed inside a control/takeover loop is
latency-bounded and latest-wins — recency over completeness; a
smoothing buffer on a control surface puts the operator behind
reality. (The opposite of a record/replay feed, where completeness
wins — pick the policy from what the feed is *for*.)

---

## 53. Works from the source checkout, fails on the compiled / installed rig

**Symptom.** A deploy / calibrate / launch path works on your dev
checkout but fails on a production rig — `python -m pkg.mod` →
"No code object available," or "vendored `scripts/…` not found,"
or a `__file__`-relative path that points nowhere. Often it breaks
an *existing* feature, not just the change you shipped.

**Looks like.** A missing file on the rig, a bad deploy, a broken
install.

**Actually is.** The code assumed the **dev source-tree layout**,
but the deployed form differs — and the side you didn't run passed
silently. A Cython/frozen package has **no `.py` and no module code
object**, so `python -m pkg.mod` can't execute the `.so` (importing
it is fine). A `parent.parent/scripts` path resolves to the
*package* dir on a source checkout (and to neither on a frozen
bundle) instead of the repo-root `scripts/`.

**Fix.**
- Invoke a deployed module by **import + entry call**
  (`python -c "import pkg.mod; pkg.mod._main()"`), never `-m`.
- Resolve paths against the **actual** deployed location; where dev
  and deployed layouts differ, **try both candidates** (repo-root
  *and* installed/bundled).
- A path/invocation that works from your checkout is **unverified**
  until it runs on the **production-built** form — exercise the
  deploy/launch paths on a compiled rig / frozen bundle, not just a
  source checkout.

See [§13.1](delivery-rules.md#dev-vs-deployed-layout).

**Generalisable rule:** the deployed artifact has a different layout
and fewer capabilities than your source tree — don't `python -m` a
compiled module, don't bake source-tree paths, and verify deploy
paths on the installed/compiled form; the untested side passes
silently and fails on the rig. (Authoritative-vs-actual cycle #1:
dev layout vs deployed layout.)

---

## 54. Runaway duplication from a non-idempotent external write / two-way sync

**Symptom.** Records pile up in an external system — a calendar
fills with duplicate events, a table grows copies — sometimes
unbounded, often right after a transient error or a restart.

**Looks like.** The external API double-firing, a webhook
re-delivery bug, the user double-submitting.

**Actually is.** A write path that **creates without first matching
an existing record by a stable id**, so every retry / reconcile
makes a new one. Classic trigger in a two-way sync: a *failed* op (a
delete that errored) leaves the reconcile pass believing the record
is missing or extra, so it re-creates the peer — and loops.

**Fix.**
- Carry your own stable id (or a content hash) on the remote record;
  **look up before create** (upsert), and key deletes/updates on
  that id, never on a mutable field (title/name).
- Make a failed step leave state the retry **converges**, not
  multiplies; cap / loudly log a reconcile that keeps finding the
  same diff.

See [§25.2](delivery-rules.md#idempotent-writes).

**Generalisable rule:** any write you might retry must be idempotent
and keyed on a stable identity — without an upsert-by-id, retries
and reconciles turn one record into many.

---

## 55. Stale because the system trusted event delivery (no reconcile poll)

**Symptom.** A push / webhook / websocket-driven integration
silently goes stale — one side stops reflecting the other — with no
error; a restart or a manual refresh "fixes" it.

**Looks like.** A flaky network, a bad webhook, the other service
dropping events.

**Actually is.** Correctness was pinned to **event delivery**, which
is best-effort: an event was dropped, missed during a restart, or
never sent for that change class — and nothing reconciles, so the
drift is permanent until a human notices.

**Fix.** Build a **periodic full reconcile poll** as the correctness
floor and let the event push only *trigger an early reconcile*
(§25.1). A missed event becomes a latency blip, not permanent drift.

See [§25.1](delivery-rules.md#reconcile-poll).

**Generalisable rule:** event push is a latency optimisation, never
the source of truth — without a reconcile poll underneath, a dropped
event is silent permanent drift. (Authoritative-vs-actual cycle #1,
across a push boundary.)

---

## 56. Web app works at localhost root, 404s behind the reverse-proxy subpath

**Symptom.** A web app is perfect on `localhost:PORT` but broken
once served through the edge under a path prefix
(`example.com/app/`): assets 404, API calls miss, links/redirects
jump to the origin root or escape the mount, the login cookie
doesn't stick.

**Looks like.** A proxy misconfig, a CORS problem, a broken build.

**Actually is.** The app **assumed it owns the origin root.**
Absolute paths (`/static/x.js`, `/api/y`, `<a href="/page">`,
cookie `path=/`) resolve at the origin, not under the proxy's
prefix — so everything off-root misses. It passed every local test
because at `localhost` root *is* the mount; the failure only shows
behind the edge you didn't run.

**Fix.**
- Reference siblings **relatively** (`./static/x.js`), or build
  every path off a **single configurable base** (`BASE_PATH` /
  `<base href>` / framework root-path) set at deploy time — never a
  constant `/`.
- Derive external URLs from **forwarded headers** / config, not the
  backend socket (behind TLS termination the app sees plain HTTP on
  a private port).
- **Verify through the edge** under the real prefix, not just at
  `localhost:PORT`.

See [§23.2](delivery-rules.md#reverse-proxy-subpath).

**Generalisable rule:** an app behind a reverse proxy doesn't own
its mount point — relative paths / one configured base, external
URLs from forwarded headers, tested through the edge; hardcoding `/`
works on localhost and 404s under the proxy. (Looks-healthy ≠
working cycle #2: green at the root origin, dead under the prefix.)

---

## 57. Parallel large transfers 502 over a contended uplink — and a tiny-sentinel probe stayed green

**Symptom.** Large uploads/transfers to a remote endpoint fail
(502 / read-timeout) almost every time, while tiny ones always
land — yet the deploy's "upload works" health probe is green. A
downstream step that needs the large object never completes (and
may cascade: data never marked uploaded → never GC'd → disk fills).

**Looks like.** A flaky server, a broken endpoint, a config error.

**Actually is.** Two compounding problems. (1) **Concurrency over a
bandwidth-bound link is net-negative**: N parallel multi-MB
transfers each get ~1/N of a thin/contended uplink, so all of them
crawl past a fixed request timeout and 502 together; serial
succeeds. A constant timeout tuned for a fast link guarantees
failure for a streamed multi-MB body over a slow one. (2) The
**probe used a tiny sentinel**, so it exercised a path the real
(large) payload never takes and passed green while production
failed.

**Fix.**
- Serialize or adapt the transfer concurrency; treat the cap as
  load-bearing (it also protects latency-critical co-traffic on the
  same link). Size the timeout to `bytes ÷ worst-case bandwidth`,
  not a constant. (§9.9.)
- Make the probe push a **representative** payload (a big-blob case),
  not a 1 KB sentinel. (§18.1.)

See [§9.9](engineering-rules.md#shared-resource-contention),
[§18.1](delivery-rules.md#smoke-probes).

**Generalisable rule:** on a bandwidth-bound link, parallelism is
net-negative and fixed timeouts lie — serialize and size timeouts
to the slow path; and a probe that doesn't carry the real payload
size is green theatre. (Looks-healthy ≠ working cycle #2.)

---

## 58. Corrupt safety-critical input passed a single-threshold gate — or "deployed" but silently fell back to defaults

**Symptom.** A model / controller behaves subtly wrong (skewed
geometry, off lane, bad estimate). The calibration/config "looks
fine," each individual number is within some limit, and the deploy
reported success.

**Looks like.** A model bug, a sensor problem, a tuning issue.

**Actually is.** A corrupt safety-critical input reached the
consumer because the **sanity gate used one threshold at a time** —
each anomaly was individually mild (1.13 aspect + 25 % off-centre),
so a `fx/fy<1.5` + off-centre-warn gate passed it. Real data has
*consistent, symmetric* anomalies; corruption *stacks independent*
ones — a per-threshold gate can't tell them apart. Or the gate
**did** reject it but the consumer silently used built-in defaults
and the deploy never verified the value was accepted on the target,
so nobody knew the calibration didn't apply.

**Fix.**
- Gate on **co-occurring** symptoms (`>15% off-centre AND
  fx/fy>1.10`, or any single extreme), not one threshold; accept a
  lone mild anomaly with a warning.
- On reject, **fail safe to a known-good default, loudly** — never
  silently feed the consumer a bad value.
- **Verify on the target**: after deploy, re-run the target's own
  acceptance gate at live conditions and report per-item
  ACCEPTED/REJECTED. "Deployed" ≠ "applied."

See [§3.5](engineering-rules.md#validate-safety-input).

**Generalisable rule:** a safety-critical input is validated on
**correlated** symptoms (corruption is the combination, not one
number), fails safe to a default when rejected, and is verified to
have taken effect on the target — one mild anomaly isn't corruption,
and "deployed" isn't "applied."

---

## 59. Shipped the wrong codebase — built from a dirty tree, or swept WIP into main

**Symptom.** `main` (or a release) contains changes nobody meant to
ship — a collaborator's half-done work, your own debugging edits —
or a deployed build behaves like code on no commit. "Where did that
come from?" with no commit to point at.

**Looks like.** A bad merge, a mystery regression, a phantom change.

**Actually is.** A shared, live checkout whose working tree held a
**mix** (your branch + collaborators' branches + uncommitted WIP),
and either (a) a `git add -A` / commit swept all of it onto `main`
to "merge everything," or (b) a build/deploy script compiled the
**working tree, not a committed ref**, baking the WIP into the
artifact. Both ship code that exists on no commit.

**Fix.**
- Merge **only your own committed, validated** branch, fast-forward,
  off-checkout (`git push origin <feature>:main`, no `--force`);
  never `add -A` WIP onto `main`. (§20.1.)
- **Commit + clean before a release build** — the `-dirty` stamp
  (§12.1) catches a build from an uncommitted tree; prefer building
  from a fresh checkout of the ref.
- Leave collaborator branches / others' WIP untouched.

See [§20.1](delivery-rules.md#merge-to-main).

**Generalisable rule:** on a shared checkout only your
committed-and-validated work reaches `main`, and a release builds
from a committed ref — an `add -A` merge or a working-tree build
ships whatever was lying around, on no commit. (Authoritative-vs-
actual cycle #1: the artifact vs any commit.)

---

## 60. The client set its own price (or role) — server trusted a posted value

**Symptom.** A user is reimbursed more than the receipt, buys at a
price you never offered, sees data their role shouldn't, or performs
an admin action from a normal account. Nothing looks wrong in the
UI; the numbers/permissions are off in the database.

**Looks like.** A calculation bug, a rounding error, a flaky
permission check.

**Actually is.** The server **trusted a value the client sent.** The
browser posted the amount / price / role / target id (or just hit an
endpoint the UI hides), and the server stored or acted on it instead
of deriving it from authoritative data. The UI is not a security
boundary — a DevTools edit or a `curl` bypasses it.

**Fix.**
- **Recompute money/authz values server-side** from the server's own
  record (the invoice's amount, the user's role); ignore the posted
  value. Bind to a **stable identity**, not a display name.
- **Authorize every privileged action on the server**, not by hiding
  the button.
- Escape rendered user content (XSS); rate-limit costly endpoints.
- For money-moving / irreversible actions add a **second approver**,
  a **cap**, and an **audit trail** (§26.2).

See [§26.1](delivery-rules.md#server-authority).

**Generalisable rule:** the client states intent, the server decides
— never trust a posted value that carries money or authorization;
recompute it from authoritative data keyed on a stable identity.
(Looks-healthy ≠ working cycle #2: the UI looked fine, the server
was the hole.)

---

## 61. Public demo wired to the production backend

**Symptom.** A "just a demo" / sandbox / trial, exposed without
login, turns out to read or write real data — or carries live
credentials — discovered when demo activity shows up in production
(or a researcher reports it).

**Looks like.** A harmless demo; "it's only the sandbox."

**Actually is.** The demo shares the production backend, database,
or credentials and is separated only by a flag or a different URL —
which a bug, an injected request, or a curious user walks around. A
public, unauthenticated surface with *any* path to prod is a breach
surface.

**Fix.**
- Run the demo as a **separate deployment** against **mock / seeded**
  data, with **no prod credentials or network reach**. Isolation by
  construction, not a runtime flag — the deliberate exception to
  §9.6's one-bundle-gate-at-runtime rule.
- Ship **zero real secrets** in a world-reachable bundle; stub any
  upstream it needs.

See [§26.3](delivery-rules.md#demo-isolation).

**Generalisable rule:** an unauthenticated public demo must be
unable to reach production *by construction* — separate deployment,
mock data, no creds — because a flag is one bug from exposing the
real system. (§26 server-is-the-trust-boundary at its limit: for a
public surface, the safest boundary is no path at all.)

---

## 62. One leaky client froze the whole stack — a capped shared resource exhausted

**Symptom.** Everything looks frozen — all consumers stop receiving
— but publishers are fine and an unrelated path (control TX) still
works. A restart fixes it; it comes back. An action that forces a
reconnect re-triggers it.

**Looks like.** The producer stack froze; a deadlock; a network
drop.

**Actually is.** A client **leaked a capped shared resource** by
re-opening it in an unbounded loop without releasing the old one —
IPC reader slots, fds, pool connections. The cap is *global to the
topic/broker*, and many brokers **evict or block all users** when
it's hit, so one leaky client takes down every consumer. Reassigning
the local handle never freed the shared slot; a restart "fixes" it
by recreating the resource (cap resets).

**Fix.**
- **Close the old handle** — don't just reassign the variable.
- **Cap reconnect attempts per silence episode** (reset on data);
  past the cap let the supervisor restart (§6.2), don't open
  unbounded sockets.
- Attribute the freeze to the **leaker**, not the frozen victims;
  ignore proxy signals (an mmap'd file's mtime doesn't move on
  write).

See [§6.4](engineering-rules.md#bounded-reconnect).

**Generalisable rule:** a capped shared resource (reader slots,
fds, connections) leaks if you re-acquire without releasing, and an
unbounded reconnect loop drives it to exhaustion — which many
brokers turn into an everyone-freezes outage. Release the handle,
cap the retries, restart past the cap. (Looks-healthy ≠ working
cycle #2: the frozen victims aren't the cause.)

---

## 63. Topic alive at the producer, silent to every new subscriber — IPC reaped on logout

**Symptom.** Consumers freeze on the value they held the moment
someone logged out of the box; probing the producer directly shows
the topic still publishing, but a *fresh* subscriber reads zero
messages; a service restart fixes it.

**Looks like.** The producer froze; a stale bridge; a dropped
connection.

**Actually is.** systemd-logind's default **`RemoveIPC=yes`**
unlinked the user's `/dev/shm/*` (POSIX shm / msgq / semaphores) at
their last logout. Live publishers keep writing their `mmap` (Linux
`unlink()` doesn't break it), but new subscribers `open()` a fresh
empty inode at the same path — producer and subscriber end up on
**different inodes**.

**Fix.**
- Don't tie a daemon to a login: run it as a **system service** or
  `loginctl enable-linger <user>` so IPC isn't reaped.
- Diagnose by inode: `/proc/<pid>/map_files | grep deleted` on the
  publisher; `stat` the shm path for a newer inode.
- Self-heal: re-bind the socket on an inode change; restart **every**
  process holding a publisher, not just one.

See [§6.5](engineering-rules.md#service-not-login-bound).

**Generalisable rule:** a user service's POSIX shm/IPC is
garbage-collected at last logout under `RemoveIPC=yes` — never tie a
long-running daemon to an interactive session; old publishers and
new subscribers split across inodes and the wire goes silent only on
the new side. (Looks-healthy ≠ working cycle #2.)

---

## 64. Consumer drops most of a fast stream — one read per tick

**Symptom.** Downstream marks data invalid (`canValid=False`,
stale), and the producer/device gets blamed ("it's silent / off"),
but the device is transmitting fine.

**Looks like.** The producer stopped; wiring; an electrical fault.

**Actually is.** The consumer reads **one item per poll iteration**,
so its intake is capped at its wake rate. Against a faster producer
it forwards a fraction and drops the rest — the low-frequency items
you actually need statistically never make it through (50 Hz × 1
frame vs a ~900 frame/s bus = ~95 % dropped).

**Fix.** **Drain all available items per wake** — a blocking read
for the first (keeps the loop idle-friendly), then a non-blocking
drain loop, batched into the downstream message; cadence unchanged.
Verify with a per-item arrived-vs-delivered count before blaming the
source.

See [§5.2](engineering-rules.md#drain-per-wake).

**Generalisable rule:** a one-read-per-tick consumer caps intake at
its wake rate and silently drops a faster producer — drain per wake,
and check delivered-vs-arrived before concluding "the source is
silent." (Looks-healthy ≠ working cycle #2: the bus was fine, the
reader was the drop.)

---

## 65. An adaptive lever re-initialized shared hardware live — wedged it under load, cascaded to everything

**Symptom.** A quality/adaptation knob (or a "reset to recover"
action) that tested fine takes down far more than its target under
real load — one subsystem's live reconfigure freezes an *unrelated*
one. Reads as the unrelated subsystem's bug, or "the network."

**Looks like.** The frozen (innocent) subsystem's fault; a flaky
link.

**Actually is.** The lever **live-re-initializes a hardware resource
shared with others** — e.g. an encoder re-cap that tears down and
re-opens NVENC, which shares the Tegra **host1x** engine with camera
capture. Under concurrent load the rapid re-init fails to re-acquire
the shared handle, **wedges the shared engine**, and cascades to
every client of it (all cameras → 0 Hz). "Verified standalone"
missed it because **one re-init at idle ≠ rapid re-inits under
load**.

**Fix.**
- Keep a **hardware-re-initializing** lever out of a live control
  loop. Prefer levers that reconfigure *without* re-init (a bitrate
  `set`, a frame-drop, a stream shed) over ones that tear down /
  re-open the device (a resolution re-cap, a device re-open); make
  the re-init thing a fixed startup setting.
- **Validate adaptation / recovery levers under realistic
  concurrent load**, not at idle (§18.1), and watch the **shared**
  resource, not just the lever's own output (§9.9).
- If a shared HW block wedges, recovery may be **re-initializing
  that subsystem's driver** (re-bind / re-train), not a reboot — a
  warm reboot may not power-cycle a co-processor (§15.7).

See [§3.4](engineering-rules.md#control-feed-freshness),
[§9.9](engineering-rules.md#shared-resource-contention).

**Generalisable rule:** a lever that live-re-initializes a *shared*
hardware resource can wedge it under load and freeze every consumer
— keep HW-re-init out of live loops, prefer no-re-init knobs, and
validate under load watching the shared resource. (Looks-healthy ≠
working cycle #2: "verified standalone" at idle ≠ safe under load.)

---

## 66. Calibration applied at the wrong resolution — geometry silently skewed

**Symptom.** A model's warp / lane overlay / projection is subtly
off — off-centre, wrong scale — but nothing errors and each
intrinsic number "looks reasonable."

**Looks like.** A bad calibration, a model bug, a camera mounting
error.

**Actually is.** The intrinsics were computed at one resolution
(full sensor, e.g. 1920-wide) and applied to a stream at another
(the served 1280-wide). `cx,cy,fx,fy` are **pixel** quantities tied
to their resolution; at a different size every term is off by the
scale factor — a centred principal point lands ~33 % off-centre.
**Silent:** the geometry skews, nothing throws.

**Fix.**
- Pin **one resolution — the consumer's (the model's)** — across
  capture, calibration, runtime intrinsics, and overlay; calibrate
  at the *served* res, not full sensor res.
- If a stage differs, **scale the intrinsics**
  (`fx,cx ×= w_t/w_c`, `fy,cy ×= h_t/h_c`) — never pass the raw
  matrix across a resolution change.
- **Store the resolution in the calibration artifact** and check it
  on load.

See [§2.2](engineering-rules.md#resolution-contract).

**Generalisable rule:** camera intrinsics / homographies are pixel
quantities valid only at their resolution — pin one across the chain
or scale them; a resolution mismatch silently corrupts geometry, it
doesn't error. (A units/contract miss, §2 — and looks-healthy ≠
working #2: every number looked fine, the frame was wrong.)

---

## 67. Build "fixed" by poking flags — green but unreproducible (or ships the wrong thing)

**Symptom.** A build was failing; someone toggled flags / re-ran /
cleared caches until it went green — but it breaks again on another
host or in CI, or it builds green and the artifact misbehaves on the
target.

**Looks like.** A flaky build, a CI quirk, "works locally."

**Actually is.** The build was debugged by **trial-and-error on a
dirty / ambient-state tree**, not reproduced from a clean committed
ref with pinned inputs. The "fix" leaned on uncommitted edits, a
host-installed lib, an env var, or a stale cache — none of it
captured — so it doesn't reproduce; or the green build was never
verified on the **deployed** form, so "it compiled" hid a runtime
failure on the target.

**Fix.** Work the §27 order: **reproduce from a clean committed tree
→ pin every input → localise the failing layer (compile → link →
package → deploy) → build for the target off a live box → verify on
the deployed form → record the fix.** A green-once build isn't fixed
until it reproduces and the artifact runs on the target.

See [§27](delivery-rules.md#build-fix).

**Generalisable rule:** a build is fixed when it **reproduces from a
clean committed tree with pinned inputs and the artifact runs on the
target** — not when it happens to go green once. (Authoritative-vs-
actual cycle #1: the green build vs a reproducible one.)

---

## 68. Peer reported "offline" / "failed" — the shared edge throttled a one-shot new connection

**Symptom.** A robot shows OFFLINE, a deploy/calibration push or an
upload "fails," intermittently and especially under load — but the
peer is up and a manual retry works. Direct/LAN it's fine; through
the cloud edge it flaps.

**Looks like.** The peer is down, a flaky network, a dead tunnel.

**Actually is.** A shared edge (reverse proxy, bastion, relay)
**throttles or saturates on *new* connections** (SSH `MaxStartups`,
a conn cap, a busy relay), and the client opened a **fresh**
connection per op with a short timeout and no retry — so the edge's
back-pressure read as a dead peer. The peer never failed; the
new-connection attempt lost the race.

**Fix.**
- **Retry with backoff** on a reachability probe before declaring
  failure — a one-shot short-timeout attempt is a false negative.
- **Ride a kept-alive / multiplexed connection** (autossh `-L`,
  HTTP keep-alive / pool) so the throttled new-connection cost is
  paid once, not per op (reuse the handle, §6.4).
- **Size the timeout to the edge path**, not the LAN (§9.9).

See [§23.3](delivery-rules.md#edge-new-conn-throttle).

**Generalisable rule:** a shared edge throttles new connections, so
a one-shot short-timeout connection false-reads back-pressure as a
dead peer — retry with backoff and reuse a kept-alive connection.
(Looks-healthy ≠ working cycle #2: the peer was fine, the edge was
the bottleneck.)

---

## 69. A second copy of critical-path logic drifted from the validated one

**Symptom.** Two paths that should behave identically — a decode, a
coordinate transform, a safety check — disagree: one is right, one
is subtly wrong, and the wrong one is on a path that matters. Each
was "correct when written."

**Looks like.** A bug in the wrong copy; a data issue.

**Actually is.** Critical logic was **reimplemented / copy-pasted
instead of reused**, so there are two sources of truth. They drifted
the moment one side changed (a fix, a tweak, a tightened threshold)
and the other didn't — **silently**, because nothing ties them
together. The validation/hardening on one copy never reached the
other.

**Fix.**
- Factor the logic into **one shared module** both callers import;
  delete the copy. Put the validation / fail-safe / hardening in
  that one place.
- Reuse the proven implementation in new contexts; don't write a
  second one "just for this."
- If a fork is genuinely needed, **record it and plan
  reconvergence** (§18.3) — never leave two silent copies.

See [§28](engineering-rules.md#reuse-critical-path).

**Generalisable rule:** critical-path logic has exactly one
implementation every caller reuses — a second copy is a
silent-divergence generator, and the wrong one eventually drives.
(Authoritative-vs-actual cycle #1: two copies of the same truth.)

---

## 70. Inherited hardware constants on a ported stack — the wrong device, silently

**Symptom.** A forked / ported perception or control stack runs
without error on new hardware, but the geometry or behaviour is
subtly wrong — an off-centre projection, a skewed warp, a model that
"works" but mis-locates things. Each constant "looks reasonable."

**Looks like.** A calibration bug, a model bug, a mounting error.

**Actually is.** The stack carried the **upstream's hardware
constants** — camera model and field of view, intrinsics, mounting
geometry, the device/sensor table, the shipped calibration defaults —
which describe the hardware the *authors* targeted, not the unit
actually installed. They load fine and the model runs, so nobody
re-derives them; but they encode a different camera (different
sensor, different FOV, a principal point and focal stored at a
resolution this stream never runs at). **Silent:** wrong device, no
error.

**Fix.**
- Treat every inherited hardware constant as a **placeholder until
  measured on the actual unit** — calibrate / measure *this* camera,
  sensor, mounting; don't trust the fork's default.
- Remember an inherited default that doesn't crash **is not
  validated** — "it loaded and the model ran" proves nothing about
  whether the numbers fit your hardware.
- **Pin each constant to its device** (store sensor / resolution /
  unit with the value) and check it against the hardware present.

See [§2.3](engineering-rules.md#inherited-hardware-constants).

**Generalisable rule:** a forked / ported stack's hardware constants
are the upstream's, valid only for the upstream's hardware — on a
different unit they load fine and silently encode the wrong device;
re-derive each from the actual unit and pin it there. (Silent-falsy /
wrong-default cycle #3: a plausible inherited default, taken for the
real one.)

---

## 71. Whole-stack crash-loop from one stale import in a gated-off module

**Symptom.** A service crash-loops — `NRestarts` climbing, exit
`status=1`, a restart every few seconds for hours; *every* subsystem
is down (no cameras, no model, nothing). It looks catastrophic — a
hardware fault, the busiest subsystem dying.

**Looks like.** A catastrophic / hardware failure; whatever changed
or runs hottest most recently.

**Actually is.** The process manager **imports every registered
module at init** (to read config / resolve entry points), *before*
any run-gate. One module had a stale `import <oldpkg>…` left over from
a rename — and even though it was **gated off and would never run**,
it's still imported, so the `ModuleNotFoundError` killed the whole
manager at startup. The blast radius was the entire table; the cause
was one import line in a process that wasn't supposed to run.

**Fix.**
- Read the manager's **first traceback** (the init exception), not
  the supervisor's restart spam — a stack-wide loop usually means the
  manager died at load, not N independent crashes.
- After a rename / refactor, sweep the old name across **every**
  registered module, not just the running ones
  (`grep -rn 'import <oldpkg>\|from <oldpkg>' …`) — a gated-off module
  is still imported (§18.2, #28).
- Isolate per-module imports (import in a `try`, disable only the
  failer, loudly) or evaluate the run-gate before importing an
  optional / dark module, so one bad module degrades to one dead
  service.

See [§6.6](engineering-rules.md#eager-import-isolation).

**Generalisable rule:** a supervisor that eagerly imports every
registered module at startup turns any one import failure into a
whole-table outage, and a run-gate doesn't protect you because a
gated-off module is still imported — sweep a rename across every
registered module, and diagnose a stack-wide crash-loop at the init
traceback, not the restart spam. (A migration survivor, #28 — and a
blast-radius misread: looks catastrophic, is one line.)

---

## 72. Destructive cleanup by glob deleted co-located bystanders

**Symptom.** A cleanup / wipe removed files it shouldn't have —
config or data that lived in the same directory as the real targets
is gone, and nothing restores it.

**Looks like.** An over-aggressive operator; a missing backup.

**Actually is.** The delete selected targets by a **broad pattern**
(`rm *.json`, `DELETE … LIKE …`, "everything in this dir") over a
**shared** namespace, so every co-located entry that matched got
swept in — not just the artifacts the operation owns.

**Fix.**
- Delete an **explicit allowlist** of owned artifacts (`{a,b,c}.json`),
  or derive the set from an authoritative manifest — never a glob over
  a shared directory.
- Treat a destructive sweep over a shared path as a §19 action; assume
  an innocent neighbour is present and scope so it can't be caught.

See [§19.1](delivery-rules.md#allowlist-not-glob).

**Generalisable rule:** a destructive cleanup deletes an explicit
allowlist of the artifacts it owns, never a broad glob over a shared
directory — a wildcard sweeps in co-located bystanders that happen to
match, and a delete doesn't come back.

---

## 73. Healthy producer, stuck consumer — a dead session's pushed override never expired

**Symptom.** A stream / encoder / consumer is pinned at a throttled
or overridden setting while everything upstream is healthy and **no
client / operator is connected**. A manual reset of the value clears
it instantly.

**Looks like.** A fault in the consumer, the producer, or the
hardware.

**Actually is.** A transient writer (a GUI session, a client, an
adaptive controller) pushed a control value through a **side channel**
(a `/tmp` file, a Param, a shared slot), and the consumer **polls the
value with no freshness check**. The writer then disconnected /
crashed / was reaped **without cleanup**, so the consumer enforces the
dead setting forever — no active sender, a ghost command.

**Fix.**
- Consumer **stamps freshness** (mtime / heartbeat / sequence) and
  **expires** a stale value (untouched > N s) to a **safe default**,
  letting the source re-adapt from a clean baseline.
- Don't rely on the writer's cleanup — a crash / reap skips it, so the
  **consumer-side TTL** is the robust half.
- Diagnose as staleness: find the side-channel value and its **age**
  before chasing the consumer.

See [§7.6](engineering-rules.md#stale-pushed-override) (and the
control-feed freshness principle, [§3.4](engineering-rules.md#control-feed-freshness)).

**Generalisable rule:** a pushed control / override value must carry a
freshness contract (mtime / heartbeat / TTL); a consumer that reads
the value without its recency obeys a dead writer forever — expire a
stale override to a safe default and let the source re-adapt.
(Looks-healthy ≠ working #2: the producer is fine, a ghost setting
drives — and a stale value read as current, #3.)

---

## 74. Hard cap blown because its eviction depended on a flaky signal

**Symptom.** A bounded store (disk, ring, cache) overran the limit it
was supposed to guarantee, or evicted unpredictably — and the cap
"should" have held.

**Looks like.** A sizing bug, a race, a too-small margin.

**Actually is.** The **must-hold cap's eviction was gated on a
best-effort / derived signal** (an upload-state xattr, a cache, a
probe) that was sometimes wrong. When it lied, the cap couldn't
enforce — the policy "protected" data it misjudged and let the store
grow past the limit. Enforcement of a hard invariant was made to
depend on a signal that can be wrong.

**Fix.**
- Enforce the hard cap on a **deterministic, self-contained key** —
  oldest-first (FIFO) over creation time plus an operator lock — never
  a fallible signal. At the cap, the oldest goes, period.
- Keep selective "spare the un-synced data" logic in the **soft /
  opportunistic** tier, where a wrong signal just frees a little less.

See [§19.2](delivery-rules.md#hard-cap-deterministic-eviction).

**Generalisable rule:** a hard invariant (cap, quota, safety limit) is
enforced by a deterministic rule over inputs you control, never gated
on a best-effort / derived signal that can be wrong — reserve
signal-dependent selection for the soft tier where being wrong is
harmless. (Authoritative-vs-actual #1: the cap trusted a signal that
diverged from the truth.)

---

## 75. Control widget showed "success" on the write returning, not on the target accepting

**Symptom.** A status indicator says the action took — "ENGAGED",
"ARMED", "mode set" — but the system is in the *other* state and
doesn't act. The operator trusts the screen; the machine disagrees.

**Looks like.** A backend / actuation bug; the wrong fingerprint;
flaky input.

**Actually is.** The widget **latched its state on the write's return
code** ("the call returned 0 → show engaged"), not on a **read-back of
the authority's actual state**. The target rejected the value
**same-tick** — a blocking safety event was active, a manual-lock
re-latched — so it never took, but the optimistic latch already
painted success. A slow reconcile poll (every 2 s) meant the lie
lingered. A safety-state indicator asserting a state the system isn't
in.

**Fix.**
- **Verify-after-write:** write, then **read the value back** (same
  round-trip if you can) and reflect the read-back; log a warning when
  it didn't take. A same-tick reject shows immediately.
- **Reconcile-poll** for *later* clears (a new block, an external
  re-latch), with a short period for safety-relevant state (§25.1).
- Same idea as [§3.5](engineering-rules.md#validate-safety-input)'s
  "verify it took effect on the target" — "issued" ≠ "applied."

See [§8.27](ui-rules.md#verify-after-write) (and
[§8.19](ui-rules.md#wire-state-mirror), the steady-state
cousin).

**Generalisable rule:** a control action confirms by a verified
read-back of the authority's actual state, never by the write
returning success — an optimistic latch lies the moment the target
rejects the value, and a safety-state indicator must never assert a
state the system is not in. (Looks-healthy ≠ working #2: the write
returned OK, the state never changed.)

---

## 76. Scanned the LAN for a rig instead of asking the registry it heartbeats to

**Symptom.** A rig went unreachable; to find it I ping-swept the
subnet and probed every live host for its hostname — slow, and it
found nothing.

**Looks like.** "I'm on the rig's LAN, so I'll just scan for it."

**Actually is.** The rig is on **DHCP**, so it doesn't own its
address: the lease moved and its *old* IP now belongs to some phone
(`.228` → a Xiaomi). A scan can't tell the rig from whatever inherited
its address, and it silently finds nothing if the rig is on another
segment. Meanwhile the rig **reports its own current `lan_ip` on every
heartbeat** — the registry already had the answer (or, here, an empty
`lan_ip` + a stale `last_seen` that *also* answered the real question:
the host is down, stop scanning and power-cycle). I reached for the
scan because I skipped the authoritative source.

**Fix.**
- **Ask the registry first:** `GET /tailnet/clients` (bearer) →
  `info.lan_ip` + `last_seen`. Then the reverse tunnel (IP-independent),
  then a direct LAN dial to *that* IP. Scan is the last resort.
- **Make the heartbeat report `lan_ip` on every stage** (factory +
  onroad), via one shared code path — an onroad heartbeat that drops
  the field empties the registry row and forces the scan you were
  avoiding.
- Read `last_seen` too: empty `lan_ip` + fresh = heartbeat bug; empty
  + stale = host down (escalate, don't scan wider).

See [§23.4](delivery-rules.md#reach-a-fleet-host-via-its-registry).

**Generalisable rule:** a host's address lives in the registry it
heartbeats to, keyed on its stable id — query that (and its freshness)
to reach it; a hardcoded IP rots when DHCP moves the lease and a scan
can't distinguish the host from whatever took its old address.

---

## 77. Wrapper script exited 0 while the work it ran failed — a false green

**Symptom.** CI is green / the scheduler logs "success" / the
supervisor shows `LastExitStatus 0`, but the actual work — a build, a
deploy, a sync — failed. Nobody noticed because the script "passed."

**Looks like.** The build/job works; the failure is somewhere else.

**Actually is.** The wrapper **didn't propagate the inner command's
exit status**. It ran the real work, then ended on an `echo`, an
unchecked pipe (`make | tee` — the pipeline's status is `tee`'s, 0),
or a swallowed `|| true` — so it returns **0 no matter what**, and
every automated consumer (CI gate, launchd, cron, a supervisor) reads
that 0 as success. We shipped it twice in a day: a build wrapper that
ended with a status line (failed build → green CI), and a scheduled
bump job whose exit was the last `echo`'s (push failures → launchd
`LastExitStatus 0`).

**Fix.**
- Capture and return the child's status: `cmd || rc=1` … `exit $rc`,
  or `set -e` / `set -o pipefail` so a failure aborts loudly. A
  trailing `echo` is usually what masks everything above it.
- Remember a pipeline's status is the **last** stage's unless
  `pipefail` is set.
- **Test the failure path:** force the inner step to fail and confirm
  the wrapper — and its CI/supervisor — goes red. An untested error
  path is dead code.

See [§18.4](delivery-rules.md#propagate-exit-code).

**Generalisable rule:** a wrapper / launcher / CI script exits with the
real status of the work it ran — never a blanket 0 from a trailing
`echo`, an unchecked pipe, or a swallowed error; a script that's green
no matter what trains every automated consumer to trust a lie.
(Looks-healthy ≠ working #2: exit 0 is a proxy; the work is the truth.)

---

## 78. A test written after the code, never seen red — it passes whether or not the code works

**Symptom.** A green test suite, but a real bug ships anyway — the test
"covering" the broken path passed the whole time.

**Looks like.** Good coverage; the bug must be somewhere untested.

**Actually is.** The test was written **after** the code and **never
watched fail** — so it encodes whatever the code already did (bug
included), asserts the wrong thing, exercises the wrong path, or is
vacuously true (`assert True`, a mock that returns the expected value,
an `await` no one checks). A test that *can't* go red proves nothing;
it's dead code that always passes — the test-layer twin of the
recovery path never seen to fire ([#23](#recurring-shapes)).

**Fix.**
- **Write the test first and run it RED** for the right reason; only
  then write the minimum code to GREEN, then refactor (§18.5). A bug
  fix starts with a test that **reproduces the bug**.
- **For a test you must add to existing code, break the code on
  purpose once** and confirm the test fails, then restore — proof it
  exercises the path.
- Watch *why* it's red: a test that fails for a setup error, not the
  behaviour, is just as blind.

See [§18.5](delivery-rules.md#test-first).

**Generalisable rule:** a test is only worth the red you've seen it
show — write it first, watch it fail for the right reason, then make it
pass; a test never seen red is dead code that asserts nothing.
(Looks-healthy ≠ working #2: a green check that can't go red is a proxy
for nothing.)

---

## 79. The test asserted on its own mock — green while the real integration is broken

**Symptom.** The suite is green, but the integrated system fails — the
component doesn't render / connect / parse in reality, and the test
"covering" it never noticed.

**Looks like.** An integration/environment problem; the unit is fine.

**Actually is.** The test **verifies the mock, not the behaviour** —
it asserts the mock was called or is present (`getByTestId(
'sidebar-mock')`), or the mock is **incomplete** (only the fields the
author thought of, so downstream code that needs the missing field
fails only in reality), or the **wrong level was mocked** (a side
effect the code depends on was mocked away, so the test passes for the
wrong reason). The mock severed the test from the thing it claims to
test.

**Fix.**
- Mock only the **slow / external boundary**, with its side effects
  understood — not the convenient high-level method.
- **Assert on real behaviour / real output**, never on the mock's
  existence or call count alone.
- Build mock responses from the **complete real structure** (captured
  or documented), not a guessed subset.
- Keep test-only methods **off** production classes.

See [§18.6](delivery-rules.md#mock-discipline) (and
[§18.5](delivery-rules.md#test-first) — watch it fail).

**Generalisable rule:** a mock is where the test stops testing
reality — keep it at the external boundary, complete, and never the
assertion target; a test that asserts on its own mock is green
regardless of whether the system works. (Looks-healthy ≠ working #2:
the mock is the proxy, the behaviour is the truth.)

---

## 80. The deployed box drifted AHEAD of the repo — the next deploy clobbered the live fix

**Symptom.** A bug that was fixed "comes back" after a rebuild /
reinstall / sync — or a deploy is diffed against the box and shows
changes nobody can account for. The live target had improvements the
repo never knew about.

**Looks like.** A regression in the new release; or tampering on the
box.

**Actually is.** A **hot-edit applied directly on the deployed
target** was never captured back to source control — a `.py` swapped
over a compiled module to recover a rig, a server file edited in
place, a feature iterated on the live copy. It worked, so nothing
forced the follow-through; the repo went quietly stale, and the next
deploy restored the committed state, **erasing work that existed
nowhere else**. (The inverse of #29 — there the target is behind the
repo; here it's ahead.)

**Fix.**
- Treat a hot-edit as a **loan**: the durable fix is committed
  source — capture it back **same day, same session**.
- **Diff the target against the repo before deploying** to a
  long-lived box; an unexpected difference is a hot-fix to capture or
  a tamper to investigate — never silently overwrite it.
- Mark live edits on the box (`.bak` alongside, a log line) so the
  next person can tell a deliberate loan from corruption.

See [§13.3](delivery-rules.md#capture-live-edits).

**Generalisable rule:** deployed-vs-repo drift runs both ways — a
target *ahead* of the repo holds work that exists nowhere else;
capture live hot-edits back to source immediately, and diff the
target before deploying so the next release doesn't clobber a quiet
live improvement. (Authoritative-vs-actual #1: the repo claimed to be
the truth while the box held it.)

---

## 81. Core daemon dies on first publish — a forgotten one-line wiring made the base class a second publisher

**Symptom.** A core process dies/crash-loops the moment some
*other* condition comes true (a bus goes live, a feature's gate turns
on) — with a publisher-collision error on a topic it shouldn't even be
publishing. Until that day, everything tested fine.

**Looks like.** A bug in the dying process; a bug in the newly-enabled
feature.

**Actually is.** A subclass **missed one wiring line** its siblings
have (`Interface = Interface`-style class attribute), so the
**framework base class** ran instead — and the base is *not inert*: it
publishes the topic ("emit empty messages for liveness"). A dedicated
publisher owns that topic; on a single-publisher transport the
late-binder dies. The collision was **latent** while the dedicated
publisher's run-gate was off, and the crash lands on the **victim**,
not the culprit.

**Fix.**
- Forgotten wiring must **fail loud** (abstract / registration error)
  or fall back to something **inert** — never to a default that
  publishes, sends, or actuates. (§7's silent-default trap in
  class-hierarchy form.)
- Every topic **names its one publisher** (§22); a collision = two
  components both believing they own it.
- **Diagnostic:** kill/observe the suspect dead, then sample the
  topics it should own — the one still publishing names the collider.

See [§5.3](engineering-rules.md#sole-publisher).

**Generalisable rule:** a base-class default that publishes is a
publisher — one forgotten subclass wiring line silently creates a
topic collision that kills the *victim* process, latent until the
other publisher's gate opens; wire explicitly, keep defaults inert or
loud, and find the collider by sampling the dead process's topics.
(Silent-falsy/wrong-default #3 — the missing attribute quietly
selected the base; and looks-healthy ≠ working #2 — the culprit looks
fine, the victim dies.)

---

## 82. Rate meter screamed an impossible number — it smoothed inter-arrival intervals through a burst

**Symptom.** A rate readout shows a number the source physically
cannot produce — "4 000 Hz" on a ~50 Hz topic — right after a stall /
reconnect / backlog drain. The operator starts debugging a fault that
isn't there.

**Looks like.** A runaway publisher, a feedback loop, corrupted
timestamps.

**Actually is.** The meter computes rate as a **smoothed reciprocal of
inter-arrival time** (EWMA of `1/Δt`). A draining backlog delivers a
burst of near-zero `Δt`s, so the reciprocal explodes and the smoothing
keeps the absurd value on screen long after the burst. The stream is
healthy (latest-wins state was never wrong) — **the meter measured its
own artifact.**

**Fix.**
- Compute displayed/alerted rates by **counting events in a fixed
  window** (`N / T`), which is burst-robust by construction.
- Treat an **impossible reading as an indictment of the meter**, not
  the stream — check the measurement math first.
- A diagnostic that misleads under edge conditions is worse than none
  (the probe-false-flag family, §10): operators panic at ghosts or
  learn to ignore it (#34).

See [§15.9](delivery-rules.md#rate-meter).

**Generalisable rule:** measure a rate by windowed event counting,
never smoothed `1/Δt` — a burst makes the reciprocal scream an
impossible number, and the operator debugs a ghost. (Looks-healthy ≠
working #2, inverted: looks-broken ≠ broken — the proxy lied, not the
stream.)

---

## 83. The fix was "in" — but it lived on a branch nobody merged, and the checkout was parked there

**Symptom.** The system is hard to track: a fix or feature everyone
assumes landed answers differently on `main`; tools and fresh sessions
see a different reality than the person who wrote the work; across
several repos, each checkout sits on a different long-lived branch and
nobody can say what "the system" currently is.

**Looks like.** Sloppy collaborators; mysterious regressions; a fleet
tool reporting stale state.

**Actually is.** Branches that were never **finished**. Work developed
on a feature branch and validated — but never merged (it strands as
the branch ages), or merged but the **checkout stayed parked** on the
branch (every tool that visits the repo reads the branch's reality),
or abandoned without deletion (it reads as pending forever). Each is
silent authoritative-vs-actual divergence at the branch level — and
unlike WIP, nothing on `main` even hints the work exists.

**Fix.**
- Every branch resolves to one of three states **promptly**: merged
  (§20.1, then branch deleted + checkout back to the default branch);
  **parked with a pushed, recorded blocker** (§18.3 — prefer the
  gated-inert merge so code lands dark instead of forking); or
  deleted.
- Make "what's unfinished?" a **command**: `git branch -vv
  --no-merged <default>` per repo — an unmerged branch with no
  recorded blocker is the finding.
- After merging, **return the checkout to the default branch** — a
  finished branch holding the checkout is how fleet tools read stale
  pins.

See [§20.3](delivery-rules.md#finish-the-branch).

**Generalisable rule:** a branch is a loan — merge it, park it with a
recorded blocker, or delete it, and put the checkout back on the
default branch; a long-lived unmerged branch with no recorded reason
is silent divergence that makes the system untrackable.
(Authoritative-vs-actual #1: main claims to be the truth while the
branch holds it.)

---

## 84. `--clean` wasn't clean — the wipe and the build disagreed about where state lives

**Symptom.** After `build.sh --clean`, some `.so` / binary still runs
yesterday's code. The operator trusts "clean means from-scratch" and
debugs the new code for a bug that lives in the stale artifact.

**Looks like.** A code bug, a broken deploy, "the fix didn't work".

**Actually is.** The clean step and the build step disagree about
where build state lives. Three mechanisms, all shipped:

- **Wrong cache path** — the wipe hardcodes one path while the build
  derives its cache per-platform (`/data/scons_cache` on one arch,
  `/tmp/scons_cache` on another). On the other arm the wipe is a
  silent no-op: the orphan sweep deletes the artifacts and the build
  *restores the stale bytes from the surviving cache*.
- **Version-keyed reinstall vs content-changed artifact** — a bundled
  / path wheel keeps the same version forever (`cereal-0.1.0`), so
  when its CONTENT changes, `pip/uv install` says "requirement
  already satisfied" and the venv keeps the old `.so` — which the
  bundler then ships. (Hash-pinned `uv sync` at least fails loud;
  `uv pip install -e .` doesn't check at all.)
- **Mtime-gated copy** — an install step using `rsync --update` (or
  mtime-keyed `package_data` staging) skips any destination whose
  mtime is newer; one hand-touched or clock-skewed file then blocks
  every future update of itself, forever.

**Fix.**
- The wipe must **derive the state location the same way the build
  does** (or wipe every arm of the per-platform split) — grep the
  build system for its cache/staging derivation before writing a
  `rm -rf` path.
- Same-version-forever artifacts get **`--force-reinstall`** (cheap,
  content-authoritative) or a content hash in the resolver path.
- Replace `--update` / mtime gates on install copies with plain `-a`
  or `--checksum` — content decides, not clocks.
- Prove a clean build the same way you'd prove any fix: change one
  line, `--clean`, and verify the **shipped artifact's bytes/behaviour**
  changed (not just that the build "ran").

**Generalisable rule:** "clean" is a contract about *state*, not a
flag: enumerate every place build state survives (compiler caches,
content-addressed object stores, venvs, staging dirs, install
destinations) and make the wipe + the reinstall path authoritative
over each — any keyed skip (version, mtime, hash-of-the-wrong-thing)
is a place staleness hides. (Cache-invalidation cycle, [#recurring](#recurring-shapes).)

---

## 85. Access granted by a shared password / copied key — can't revoke one person, can't audit who

**Symptom.** An operator left months ago but might still have access;
nobody's sure who can reach the box; a single leak (a pasted password,
a private key copied to a laptop) would force rotating the credential
for *everyone* and re-distributing it — so it never happens, and
access sprawls.

**Looks like.** A people-process gap; "we'll clean it up later."

**Actually is.** Access was provisioned with a **shared secret** — one
password or one private key handed to many — instead of each person's
**own public key** on a least-privilege account. A shared secret can't
be revoked per-person (rotation hits everyone), can't attribute who
acted (one identity for all), and has to *travel* to each user (every
hop a new exposure); one leak is total compromise.

**Fix.**
- Authorize the requester's **own public key** on a **least-privilege
  account** (forwarding-only / read-only), behind an approval gate
  (§26.2). Revoke = delete one `authorized_keys` line.
- Never DM a password or copy a private key to a second machine — a
  public key is public, so nothing secret is transmitted.
- Scope the account to exactly the role's need so a compromised key
  can't exceed the grant.

See [§26.4](delivery-rules.md#per-identity-access).

**Generalisable rule:** grant access by the requester's own public key
on a least-privilege account behind an approval gate — never a shared
password or private key; per-identity credentials revoke and audit
individually, a shared secret can't be revoked for one person and
forces a fleet-wide rotation on every departure or leak.

---

## 86. A "service" that's really a cron one-shot — idle between fires, never starts on boot, never self-heals

**Symptom.** A job meant to keep something up-to-date / always-on shows
as **idle** every time you check it; after a reboot it doesn't run until
its next scheduled time; when it dies nothing brings it back — and
operators read "idle" as "broken."

**Looks like.** A dead or misconfigured agent.

**Actually is.** It was built as a **scheduled one-shot** (a cron line,
a systemd `*.timer`, a launchd `StartCalendarInterval`) when the
requirement was a **resident service**. A one-shot has no live process
between fires: it can't start on boot (it waits for the next clock
match; a fire missed while off/asleep is skipped), there's nothing alive
to keep alive (no self-heal), and its normal resting state is "no PID" —
which looks dead.

**Fix.**
- If it must be always-on / boot-started / self-healing, run it as a
  **resident loop** (`do-work → sleep → repeat`) under a supervisor with
  **start-on-load** (`RunAtLoad` / `enable`) + **restart-on-exit**
  (`KeepAlive` / `Restart=always`). A failed cycle logs and continues;
  only a crash exits, and the supervisor restarts it.
- If a missed run really is harmless, keep the one-shot — but then
  **"idle" is the healthy state**, so don't alarm on it.
- Verify the live state, not the config: after install it shows
  **RUNNING with a PID**; kill it and confirm the supervisor respawns it.

See [§6.7](engineering-rules.md#resident-vs-scheduled).

**Generalisable rule:** match the lifecycle to the requirement — an
always-on job is a resident process under start-on-load + restart-on-exit
supervision, not a calendar/cron one-shot; a one-shot has no process
between fires, so it can't boot-start or self-heal and its idle state
reads as dead.

---

## 87. A bootstrap that writes the files but doesn't commit — the next tool can't see it, the manual finish is forgotten

**Symptom.** You ran the one-command installer / bootstrap, but the repo
isn't actually set up: the managing tool reports it as not-tracked /
stale / broken, and only a half-state is on disk — staged files, an
uncommitted pointer, untracked symlinks.

**Looks like.** A flaky or broken installer.

**Actually is.** The tool stopped **one step short of the handoff**: it
wrote/staged the artifact but didn't **commit** it (or built-but-didn't-
push, generated-but-didn't-register). The next tool in the pipeline reads
the *committed / pushed / registered* state, not files-on-disk — so it
can't see the work, and the "now run `git add && commit` yourself" note
was forgotten.

**Fix.**
- Make the tool **finish**: commit the artifact (push it, register it) so
  it lands in the exact state the downstream tool manages.
- The finishing commit is **path-limited** — stage an explicit allowlist
  of the files the tool owns (`git add -- …`) and commit those paths,
  **never `git add -A`** (which would sweep co-located WIP).
- Make it **idempotent**: a re-run with nothing to do commits nothing.

See [§18.7](delivery-rules.md#finish-the-operation).

**Generalisable rule:** an automating tool is done at the
committed/pushed/registered state its downstream consumer reads, not when
the files are written; finish the handoff with a path-limited commit (no
`add -A`) and keep it idempotent.

---

## 88. "Already up to date — nothing to do," but the last push failed and no re-run retries it

**Symptom.** A sync / bump / deploy tool reports the target is "already
at X — nothing to do," yet the consumer still sees the old version — and
no number of re-runs fixes it. You push the stranded commit by hand and
it's fine.

**Looks like.** The tool ran and said done — must be a consumer-side
cache, or someone reverted it.

**Actually is.** The tool's "skip, already done" check compares a
**local** marker (the committed pointer, a written file) to the target,
not the **published** state. A prior run committed/wrote locally but its
**push/apply failed** (a flaky link, an auth blip, a killed fan-out). The
commit is local, so every later run sees `local == target` and skips —
the unpushed delta is stranded forever, invisible to the re-run that's
supposed to heal it.

**Fix.**
- Gate "done" on the **consumer-visible end-state**: `local == remote`
  (committed **and** pushed), the *applied* value, the *served* artifact
  — not local progress.
- On a failed publish, leave it **detectably not-done** so the next run
  retries (don't record success on a partial run).
- When a push fails mid-fan-out, **re-push that stranded commit
  directly** — a tool that skips "already committed" targets won't.

See [§18.7](delivery-rules.md#finish-the-operation).

**Generalisable rule:** an idempotent tool's "skip, already done" test
keys on the consumer-visible end-state (pushed / applied / served), not a
local marker — else a prior run that finished locally but failed to
publish is declared done and its tail is stranded across every re-run.

---

## 89. A one-shot operator action strands a half-done state — re-clicking errors or double-applies, so recovery needs a CLI

**Symptom.** An operator runs a GUI action (onboard, link, enroll,
deploy); it half-completes or fails; pressing it again **errors**
("already exists" / "already linked") or **double-applies** — so the only
way out is a terminal and someone who knows the CLI.

**Looks like.** A flaky one-off action; "just SSH in and finish it."

**Actually is.** The action was built as a **one-shot**, not an
idempotent converging flow: it assumes a clean start, doesn't check what
each step already achieved, and has no safe re-run. A first-attempt
failure (a dropped push, a same-tick reject) leaves a half-done state the
button can't heal — and a field operator has no CLI.

**Fix.**
- Make the action **idempotent**: each step checks its **end-state**
  (read-back, §8.27) and does only what isn't done; "already done" is a
  clean no-op, not an error and not a second apply.
- Make **re-run the recovery path** — one click, always available (don't
  hide it or require "undo first"); label it for what it'll do
  ("Re-link", "Retry").
- Report the **verified end-state** and which step is still outstanding,
  so the re-press is informed, not blind.

See [§8.29](ui-rules.md#rerunnable-action); the tool / CLI form is
[§18.7](delivery-rules.md#finish-the-operation) (anti-patterns #87, #88).

**Generalisable rule:** an operator action that drives a multi-step flow
is idempotent and re-runnable — a second press converges (skips done,
retries failed) and is the one-click first-line recovery — never a
one-shot that strands a half-done state fixable only from a CLI.

---

## 90. Hand-rolled a one-off viewer to debug a sensor / perception pipeline — N disconnected tools, no replay

**Symptom.** To see what a perception / robotics pipeline is doing you
built a bespoke viewer — a matplotlib dump here, a custom 3D widget
there, a print of tensor shapes, an ad-hoc image window. Each is
separate, none share a timeline, and once the run is over there's
nothing to replay.

**Looks like.** "I just need to *look* at the point cloud / detections
real quick."

**Actually is.** Reinventing a multimodal, time-aware viewer — badly.
The modalities (3D, images, scalars, tensors, transforms) don't
correlate on one clock, you can't scrub time, and there's no recording
to inspect after the fact, so every debug session starts from scratch.

**Fix.**
- Log to **Rerun** (`rerun-io/rerun`): `rr.log(entity_path, …)` for
  images / point clouds / boxes / transforms / scalars / tensors, and let
  its viewer render, scrub, and correlate on the shared clock (§2.1).
- View live (`connect_grpc` to a running viewer) or replay the canonical
  unomsg `.log` into a viewer — but **never `rr.save(".rrd")`**: a `.rrd`
  is a *second* record beside the log of record and drifts (§28 two-copies
  trap). The diagnostics artifact (§15) is the `.log` (§16.9), not an
  `.rrd`; Rerun is the lens, not the record.
- Keep operator/control surfaces in the house GUI — Rerun is the
  introspection layer, not the control layer.

See [§8.30](ui-rules.md#rerun-visualization).

**Generalisable rule:** for multimodal, time-indexed telemetry / debug
visualization, log to Rerun and let its viewer render-scrub-replay on one
timeline — don't hand-roll a one-off viewer; reach for the control GUI
only to *operate*, Rerun to *see and debug*.

---

## 91. A dropped connection mid-push/deploy read as "failed" — but the write had landed; the blind retry wasted or duplicated it

**Symptom.** A push / deploy / upload / API write errors with
"connection closed," "broken pipe," a sideband disconnect, or a verify
that timed out. You treat it as failed and retry — and it says "already
up-to-date / already exists," or (if not idempotent) you get a duplicate.

**Looks like.** The operation failed; run it again.

**Actually is.** On a flaky / throttled link the server **applied the
change, then the channel dropped before the ack** — or it was the
*verify* step, not the write, that dropped. The error is on the
confirmation channel, not the operation.

**Fix.**
- Before retrying, **re-verify the end-state** with a cheap idempotent
  read (`git fetch` + `local == remote`, a `GET`, the row's value) and
  retry only what genuinely didn't take.
- Make the write **idempotent** (upsert by stable id §25.2, `push`,
  `PUT`) so a redo converges, never duplicates.
- Don't declare "peer offline / failed" from one dropped connection —
  the verify channel is as flaky as the write channel (§23.3); re-probe
  with backoff.

See [§23.5](delivery-rules.md#reverify-before-retry).

**Generalisable rule:** a mid-operation transport drop or lost ack is not
proof the write failed — re-verify the consumer-visible end-state with an
idempotent read before retrying, and make the write idempotent so a redo
converges.

---

## 92. A "keepalive publisher" on an input topic silently assassinated the real producer — looked like a flaky transport fault

**Symptom.** A daemon works alone but, in the full system, dies with
`transport: send failed (-1)` / `Closed` shortly after some *other*
process starts — and *which* process dies depends on start order /
timing, so it reads as a flaky transport bug.

**Looks like.** A flaky IPC / transport fault; a race in the messaging
layer.

**Actually is.** On a single-publisher, **claim-on-create** transport,
constructing a publisher stamps a fresh writer-id into the topic segment
and **revokes the previous publisher**. A process had opened a
**keepalive publisher** on a topic it only *consumes* (an old pattern to
make `subscribe` work before transports could attach-or-create) — so
whichever process opens it *after* the true producer silently kills the
producer's next send.

**Fix.**
- **Never create a publisher on a topic you don't send on.** Subscribers
  attach-or-create the segment; the keepalive publisher is obsolete and
  harmful.
- Diagnose by the signature: a process dies `send failed (-1)` right
  after another starts ⇒ a publisher collision on one of its *output*
  topics (§5.3 — sample the dead process's topics).
- Size the ring for the largest message (`max_msg_bytes` ≥ the biggest
  payload) — unrelated to the collision, but the next thing that bites.

See [§5.3](engineering-rules.md#sole-publisher).

**Generalisable rule:** on a claim-on-create single-publisher transport,
opening a publisher claims the topic even if you never send — so never
create a publisher on a topic you only consume (subscribers
attach-or-create); a keepalive publisher assassinates the real producer
in a start-order-dependent way that masquerades as a flaky transport
fault.

---

## 93. A plan / selection that recomputes every tick flaps between near-equal options — downstream chases a jittering reference

**Symptom.** A periodically-published decision — a route, a plan, a
chosen target, a mode — flip-flops constantly even though nothing
meaningful changed; whatever consumes it (a controller, a UI, a worker)
chases the jitter and never settles.

**Looks like.** A noisy sensor, a flaky downstream, "the planner is
unstable."

**Actually is.** The loop **recomputes the decision from scratch every
tick.** A non-trivial solver returns a slightly different answer from
each slightly-shifted input, so re-solving every period yields a
different near-equal solution each time — the oscillation is *self-
inflicted by re-deriving instead of holding*.

**Fix.**
- Each tick, **re-validate the current decision** ("is it still good?")
  rather than recomputing ("what's optimal now?"). Keep it while valid.
- **Recompute only on a real trigger** — it became invalid, a genuinely
  new input arrived, or a failure *persisted* (a transient blip the
  downstream recovers from doesn't force a change).
- **Add hysteresis** — a new candidate must be meaningfully better (a
  margin / debounce / min-dwell) before you switch.

See [§3.6](engineering-rules.md#decision-hysteresis). Runtime cousin of
recurring shape #6 (decision oscillation) and §9.7 (accelerator
oscillation).

**Generalisable rule:** a periodically-recomputed decision oscillates
when re-derived from scratch each tick — re-validate and hold the current
one, recompute only on a real trigger, and require a margin before
switching; applies to plans, routes, selections, leader election,
autoscaling, and adaptive quality alike.

---

## 94. Per-consumer adaptive loops over one shared cap climb together and collapse — budget the total, divide it

**Symptom.** An adaptive system over a shared, capped resource (video
streams on one uplink, clients on one pool/limiter) **flaps on a ~minute
cycle**: it sheds consumers, recovers, over-commits, collapses, sheds
again — forever — even though the cap itself is steady.

**Looks like.** A flaky link / a noisy resource / "just needs better
tuning."

**Actually is.** Each consumer runs its **own independent** adaptive
(AIMD / backoff) loop. At low utilisation every loop reads its *own*
queue as "clear" and probes up — into the **same** shared headroom the
others are also claiming — so they climb together and **multiply demand**
past the shared cap → collapse → all back off → repeat. One consumer's
"room" is another's contention; independent loops can't see that.

**Fix.**
- Run **one controller on the *total* budget**; set each consumer's share
  = budget ÷ active consumers. **Adding** a consumer scales every share
  *down*, never up — a restore can't multiply demand.
- Concurrent sessions divide the *same* budget (don't give each its own).
- **Dead-band + asymmetric smoothing** (fast-down on collapse, slow-up to
  ride the AIMD ripple) so it doesn't hunt at the shed/restore boundary;
  one writer owns the throttle (§5.3).

See [§3.4](engineering-rules.md#control-feed-freshness).

**Generalisable rule:** adaptive allocation of a shared, capped resource
closes the loop on the **total** budget and divides it among consumers —
N independent per-consumer loops each probe into the shared headroom,
overshoot together, and oscillate; budget the total so adding a consumer
shrinks every share instead of multiplying demand.

---

## 95. New code shipped, but its dependency was hand-installed where the build wipes it — silent degrade on the next build

**Symptom.** A deploy worked; then one rebuild / onboard later the feature
is silently running on a **fallback / defaults** — no crash. The new code
is present, but the capability it needs is gone.

**Looks like.** A flaky feature, a config that "reset itself," a
regression with no commit behind it.

**Actually is.** The change shipped in **two parts**: code in a durable
spot (outside the rebuilt tree) and its dependency **hand-installed into a
build-regenerated location** (a venv's `site-packages`, `target/`, a
cache). The next build recreated that location and **wiped the dep**; the
code survived and fell back. The build "succeeded," so nothing flagged it.

**Fix.**
- Install the dependency through the **build's own manifest / lockfile**
  (or bootstrap install phase) so every rebuild reinstalls it — not by
  hand into the regenerated tree.
- Ship code + its new dependency as **one paired unit** through the
  repaved path; never a durable half + a wiped half.
- Verify on the **rebuilt** form by **loading** the dep (a CRITICAL log
  on missing, not a silent fallback).

See [§13.4](delivery-rules.md#regenerated-location).

**Generalisable rule:** anything hand-placed in a build-regenerated
location is erased by the next build — install runtime deps through the
build's manifest, deploy a code-plus-dep change as one paired unit, and
verify by loading on the rebuilt form, so a half-reverting deploy can't
silently degrade.

---

## 96. A new subscriber wasn't in the bridge's hand-maintained forward list — data never crossed, blamed on the source

**Symptom.** A consumer (a GUI panel, a downstream service) shows **no
data** — not an error, just blank / "daemon silent" — for a source that
is **healthily publishing**. The fault gets chased on the *source* (a
dead daemon, a broken sensor) when the source is fine.

**Looks like.** A dead producer / a flaky link / a sensor fault.

**Actually is.** The bridge / forwarder / router serves a set derived
from a **hand-maintained parallel list** (a tuple of subscriber names, a
rebind table), and the new component was added but **not added to the
list** — so it's silently never forwarded. Nothing errors; the data just
never crosses. (Shipped twice — `liveTracks`, then `parkingState`.)

**Fix.**
- **Derive the served set by type / interface**, not a name-list:
  `isinstance` the subscriber base, scan the plugin registry, reflect the
  handlers — so a new component is included **automatically** and can't
  be forgotten.
- Make inclusion **structural**, not a checklist step you re-apply.
- When a consumer shows no-data for a healthy producer, check the
  **bridge/router forwards it** before blaming the source (anti-pattern
  #1).

See [§5.4](engineering-rules.md#derive-the-registry).

**Generalisable rule:** a bridge / forwarder / registry derives the set
it serves from the live components (by type or interface), never from a
hand-maintained parallel list — a second list drifts when a component is
added, drops it without erroring, and the missing data is misattributed
to the source.

---

## 97. Builds clean on the dev host, breaks on the target — a target-specific bug the host build hid

**Symptom.** Code compiles and runs fine on the dev machine, but on the
target architecture it fails to build, links wrong, or misbehaves at
runtime — a struct that's the wrong size, a `#[cfg]` / `ifdef` branch
never compiled on the host, a syscall / frame layout that differs by arch.

**Looks like.** "Works on my machine"; a flaky target; a deploy problem.

**Actually is.** The **host build is not representative** of the target's
`(arch, libc, OS)`. Target-only code paths (conditional compilation,
arch-dependent struct / syscall layout, libc / kernel-header features) are
never exercised by the host compiler — so the host build is green while
the target is broken, and you find out at deploy or in the field.

**Fix.**
- **Cross-build to every target early and continuously** (CI / pre-merge,
  not just at release) — the target build is a verification surface that
  catches what the host build silently passes.
- Drive it from a **declared target matrix** + one reproducible
  toolchain-activation command (§11.3), so "build for the rig" is one
  command, identically, on any host.
- Then **run on the target** too (§13.1) — building for the target arch
  still isn't the target *running* it.

See [§11.3](delivery-rules.md#cross-build-matrix).

**Generalisable rule:** the host build is not representative — target-only
code (cfg branches, ABI / struct layout, libc features) compiles clean on
the host and breaks only on the target's triple; cross-build to every
target early and often as a verification surface, then run on the target.

---

## 98. The control link dropped but the actuator kept obeying the last command — runaway

**Symptom.** An operator disconnects / the network drops mid-teleop, and
the robot keeps driving on the **last** joystick command (held throttle,
frozen steer) instead of stopping. The longer the link stays down, the
longer it coasts on a dead command.

**Looks like.** "The controller is fine — it's doing exactly what it was
last told."

**Actually is.** The command consumer **actuates on the last received
value with no freshness check**, so a dead/stale link is indistinguishable
from "the operator is holding the stick." It replays the stale command for
a hold window (or forever). Doing what it was last told is a **runaway**
when nobody is sending anymore.

**Fix.**
- Check **command freshness every tick** (the command's timestamp / a
  sender heartbeat — **unmaskable**, not a `connected` flag a dead sender
  leaves `True`); past a small staleness bound, bail to a **validated safe
  state** (idle / coast-to-stop / neutral / brake).
- Centralize it in a **deterministic watchdog** (`Live` / `SafeStop` /
  `EStop`); the actuator reads the watchdog state, not raw axes.
- Recover automatically when fresh commands resume.

See [§3.7](engineering-rules.md#command-link-safe-stop). Command-direction
twin of §3.4 (display-feed freshness); safety-critical case of §7.6.

**Generalisable rule:** a link that actuates safe-stops when its command
goes stale — check command freshness on an unmaskable signal every tick
and bail to a validated safe state past a small bound; never keep
replaying the last command after the link drops, because a held command is
a runaway.

---

## 99. The binary only runs with `LD_LIBRARY_PATH` set — "image not found" everywhere else

**Symptom.** A binary runs from your dev shell but dies elsewhere (a
systemd unit, cron, another operator, the rig) with `image not found` /
`cannot open shared object file: No such file or directory` — and the
"fix" is to `export LD_LIBRARY_PATH` / `DYLD_FALLBACK_LIBRARY_PATH` before
every launch.

**Looks like.** A missing dependency / a broken install.

**Actually is.** The binary has **no baked rpath**, so it can only find
its shared lib when an env var happens to point at it. That env lives
*outside* the artifact — so anywhere it isn't set (a service, cron, a
clean shell, the deployed target) the loader can't find the lib. The
artifact isn't relocatable.

**Fix.**
- Link with a **relative rpath** (`$ORIGIN/../lib` / `@loader_path/../lib`)
  so the binary + its libs are relocatable and resolve with no
  environment.
- `patchelf` / `install_name_tool` is a fallback, not the plan — build it
  relocatable so there's no post-hoc step to forget.
- **Verify from a clean shell** (no `LD_LIBRARY_PATH` / `DYLD_*`) on the
  deployed layout: `ldd` / `otool -L` resolve every lib via the rpath.

See [§11.4](delivery-rules.md#relocatable-rpath).

**Generalisable rule:** a deployed binary finds its shared libs through a
baked relocatable rpath (`$ORIGIN` / `@loader_path`), never a runtime
`LD_LIBRARY_PATH` / `DYLD_*` env var (an env-var crutch, §7.5) or a
forgotten `patchelf`; verify by running from a clean environment on the
deployed layout.

---

## 100. The control loop hunts because it closes on a proxy that decoupled from the real plant

**Symptom.** An actuator command oscillates at a near-constant frequency
(a ~1 Hz limit cycle) — current / torque pulsing — while the plant barely
responds; the state estimate shows impossible values (accelerations the
platform can't produce).

**Looks like.** A noisy sensor, a bad estimator — "tune the filter / lower
the gain."

**Actually is.** The controller closes its loop on a **proxy** for the
variable it controls (motor rpm as a stand-in for ground speed), and in
this regime the proxy **decoupled** from the true quantity (drivetrain
slip — the motor spins, the body doesn't). The loop chases the proxy; the
plant doesn't follow; it hunts.

**Fix.**
- Close on the **true controlled variable** (a fused ground speed, not
  motor rpm).
- If a proxy is unavoidable, **cross-check it against an independent
  signal** (IMU / GPS) and **fail safe** (§3.5) when they diverge — a
  plausibility gate catches the limit cycle, but sensing the real variable
  is the cure.
- A hunting actuator with the plant *not* responding is a **feedback**
  problem (what it closes on), not the output filter.

See [§3.8](engineering-rules.md#proxy-feedback).

**Generalisable rule:** a feedback loop closes on a faithful measurement
of the variable it controls — a proxy that decouples from the true
quantity under some regime (slip, saturation, compliance) makes it hunt;
sense the real variable, or cross-check the proxy and fail safe when they
diverge.

---

## 101. Edited the shared lib's internal workspace, but consumers depend on its published client — the change never shipped

**Symptom.** You change a shared cross-language lib, rebuild, re-publish,
re-pin the consumer — and the consumer still behaves the old way. Or a
consumer launched from a different directory dies with `UnknownService` /
a missing-symbol error the lib "definitely has."

**Looks like.** A stale pin, a caching problem, a flaky build.

**Actually is.** Two shapes of the same cross-language-lib trap:
1. The lib splits into an internal `crates/` workspace **and** a thin
   `client/` that consumers actually depend on (the publish mirrors only
   `client/`). You edited the workspace; the published client — what
   consumers link — never changed.
2. The client resolves its config/registry table **relative to the CWD**
   (`./config/…`), so a consumer started outside its repo root silently
   falls back to the dylib's *embedded* table.

**Fix.**
- Ship the **published surface** consumers depend on (the `client/`
  subtree), bump its version, re-publish; **verify by building a
  consumer** against the new prefix, not the lib's own tests.
- Know your change's **ABI class**: additive needs no reship; a wire /
  header / format bump reships every triple + re-pins every consumer
  (§28.1, §4).
- Make config resolution **CWD-independent** — register the table
  explicitly at init (or embed it), don't read `./config/…`.

See [§28.2](engineering-rules.md#cross-language-lib).

**Generalisable rule:** a cross-language shared lib publishes a stable
narrow ABI + a thin client and keeps the schema on the consumer side;
ship the published client surface (not the internal workspace), classify
the change's ABI impact before publishing, and resolve config independent
of the CWD.

---

## 102. A silent duplicate of a singleton process — both write the shared state, the system goes stale / flaps

**Symptom.** A setting won't hold, or a shared value alternates between two
near-values, or a stale config keeps driving the system — and a restart
"fixes" it briefly. The producer looks healthy; the *consumer* gets
blamed.

**Looks like.** A flaky consumer, a stale cache, a config that "resets
itself."

**Actually is.** Two instances of a process that should run **once** are
running at once — a relaunch that didn't stop the old one, an operator who
ran it again, a session that exited without cleanup, a double-spawn. They
don't crash; both **write the same shared state** (a topic, a `/tmp` slot,
a throttle), which now flips between their outputs. (Shipped as a dead GUI
session's leftover throttle, two viewer sessions on one budget, a keepalive
publisher vs the real one.)

**Fix.**
- **Guard single-instance at startup** — a `flock`'d pidfile / `O_EXCL`
  lockfile / bound singleton socket / the topic claim itself (§5.3). A
  second start no-ops or takes-over-and-reaps; it never silently coexists.
- **Reap the predecessor / orphans** on startup, and **expire stale
  pushed values** on the read side (§7.6) so a leaked duplicate can't
  drive forever.
- **Diagnose by counting writers** — `pgrep` / `ss` the instances and the
  topic's publishers before tuning the value or blaming the consumer.

See [§6.8](engineering-rules.md#singleton-process).

**Generalisable rule:** a process that must run once per host/resource
guards single-instance (lock/claim, take-over-and-reap) and reaps the
predecessor — a second silent instance writes the same shared state and
the system goes stale/flaps, with the symptom landing on the victim, not
the duplicate.

## 103. Mixed versions of a built lib — the source says one thing, the bytes on disk say another

**Symptom.** A fix / feature is "in the lib," you rebuilt and re-pinned,
but a consumer still runs the old behaviour — or different machines /
arches behave differently for the *same* pin. You can't tell which version
is actually loaded.

**Looks like.** A stale pin, a caching problem, "works on my machine."

**Actually is.** The rebuild was **partial** — one triple rebuilt and not
the others, a fresh dylib for one arch over stale ones, a re-publish that
bumped the source tag but not every artifact — so the prefix holds **mixed
versions**. The source (`Cargo.toml` / `pyproject`) says the new version;
the bytes on disk are older. A shared, non-version-namespaced dylib hides
it: same pin, different bytes per machine.

**Fix.**
- Make the artifact **self-report its version** (an exported
  `…_version()` symbol, an embedded string, a sidecar manifest, with the
  `git_sha` — §12.1), and **verify the version that's loaded/linked**, not
  the source you're looking at.
- **Rebuild + publish every triple from one version atomically** (§11.3,
  §28.1) — never some arches and not others.
- Read the version from **one source** into the stamp, the tag, and the
  manifest.

See [§28.3](engineering-rules.md#lib-version-stamp).

**Generalisable rule:** a built lib carries a queryable version stamp and
you verify the version actually loaded/linked (not the source), and you
rebuild every triple from one version atomically — partial rebuilds across
a shared prefix are the default way built-lib versions get mixed up.

---

## 104. A clean rewrite drifted back into the legacy it replaces — legacy names / layout / identity crept in

**Symptom.** A "from-scratch rewrite" keeps acquiring the old system's
shape — types named after the old classes, a directory tree mirroring the
legacy, the upstream project's name in symbols / logs, an `import` of the
old stack "just for now." The new system is becoming the old one in a new
coat.

**Looks like.** Pragmatic reuse / "porting faster" / "less divergence."

**Actually is.** A rewrite re-assembles known pieces, so without a
recorded **clean-break** decision the legacy's naming, structure, and
identity creep back in and **re-couple** the new system to the very thing
it was meant to replace — and every later change then has to fight the
legacy's shape.

**Fix.**
- New **namespace + conventions throughout** — internal modules, types,
  layout, CLI, log tags — not just user-visible strings (§1).
- Depend on the legacy **only at one deliberate, narrow seam**: share the
  **schema as data** (lineage-guarded, §28.2), never the legacy's code /
  types / build. Wire-compatible ≠ code-coupled.
- **Re-state defaults** from your own source (§2.3); **record the
  clean-break decision** (§18.3) so a reviewer can call drift what it is.
- **The tell is the carried *names*, not the comments** — legacy
  daemon/process names (a `*d` set), topic/field names, module/type
  names, config paths recreated one-for-one. Rename those; a **provenance
  comment** ("ported from `<legacy>`'s X") is *exempt* (§1) — keep it,
  it's lineage, not a fork.

See [§1 — clean rewrite](engineering-rules.md#rewrite-clean-break).

**Generalisable rule:** a ground-up rewrite is a clean break, not a fork
— carry the new namespace / structure / identity all the way through,
depend on the legacy only at one narrow interop seam (schema as data, not
code), and treat drift back into the legacy's names/layout/identity as a
regression.

---

## 105. Two versions of the same shared lib in one graph — "expected Service, found Service"

**Symptom.** A build fails with a type mismatch where the two types are
**literally the same name** ("expected `unomsg::Service`, found
`unomsg::Service`"), or a dylib crashes with duplicate symbols, or a
consumer's value silently never matches the producer's.

**Looks like.** A baffling compiler bug — "they're the *same* type!"

**Actually is.** Two different **versions** of the shared lib are in the
dependency graph — different pins across crates, or a transitive dep
pulling a second version. Same-named types from different lib versions are
**distinct types** and don't unify; two C-ABI copies in one process are
undefined behaviour.

**Fix.**
- Pin the shared lib **once** (a single workspace dep the tree inherits),
  and **align every consumer** to the same version (fleet alignment,
  §28.1) — collapse any diamond to one version.
- Use the **version stamp** (§28.3) to find which consumer resolved the
  stray version.

See [§28.4](engineering-rules.md#single-version-graph).

**Generalisable rule:** a shared lib that defines boundary-crossing types
resolves to exactly one version across a build's graph — pin it once,
align every consumer, and collapse a second version (its "identical"
types don't unify; two C-ABI copies are UB) rather than shipping two.

---

## 106. Every log line carries its source twice — the framework's target plus a hand-written `[tag]`

**Symptom.** Log lines read with the component name (or the level, or the
timestamp) **duplicated**:
`[2026-… INFO selfdrived] [selfdrived] carState 50Hz`. Across the fleet
the format is also inconsistent — some binaries print one shape, their
siblings another.

**Looks like.** A cosmetic logging glitch; "the prefix is just doubled."

**Actually is.** The line is being **formatted twice**. The logging
framework's formatter already stamps `timestamp + level + target`, and
the *target is the component identity* (the crate/module/logger name) —
then the code *also* hand-prepends `[component]` in every message, so the
identity lands twice. The siblings differ because one calls the
framework's bare `init()` (default format) while another builds a custom
formatter — and a library that installs its own logger stacks a second
formatter on the same record. The same doubling hits ts/level when a
supervisor re-logs a child's already-formatted stdout, or when an
already-rendered string is run back through the formatter.

**Fix.**
- Pick **one owner of the prefix**: either let the framework emit
  ts/level/target and write *bare* messages, or disable the framework's
  target (`.format_target(false)` / drop `%(name)s`) and own the tag —
  never both.
- Configure the format in **one shared init** every binary calls; a
  library logs through the facade and installs **no** formatter.
- A supervisor forwards a child's captured output **verbatim** (or the
  child logs straight to the shared sink) — it never re-stamps a line
  that's already a log line. Format the record once at emit; pass
  structured fields, not pre-rendered text.

See [§16.8](delivery-rules.md#format-log-once).

**Generalisable rule:** a log line is timestamped, levelled, and prefixed
with its source exactly once by a single owner — the framework already
stamps ts + level + target, so don't hand-prepend the same `[component]`
tag, don't stack a second formatter (a divergent per-binary init or a
library that installs its own), and don't re-log a child's already-
formatted output at a supervisor.

---

## 107. The status badge says "down" while the data is streaming — it polls a side-channel probe, not the data link

**Symptom.** A health/stack/connection indicator reads **down /
unreachable / stopped** at the very moment the system is demonstrably up —
its real telemetry (carState, frames, metrics) is flowing on screen. The
badge also **flickers** between up and down poll-to-poll.

**Looks like.** A flaky service / a transport fault on the rig.

**Actually is.** The badge asks a **separate, out-of-band probe** "is it
up?" — a fresh SSH handshake / HTTP healthcheck / liveness ping on its own
connection — while the payload streams on a *different*, persistent one.
The data tunnel stays warm because it's actively streamed; the probe's
connection sits **idle between polls**, so a shared edge drops it and
re-handshaking is throttled (§23.3) → the probe false-reads "down"
forever. Two sources of truth for one question, and the flaky one drives
the badge.

**Fix.**
- The producer **publishes its own state as telemetry on the primary data
  link** (a supervisor's process set, a daemon's `live` + last-update
  stamp) — status is a topic, not an out-of-band question.
- The consumer **derives up/down from telemetry liveness**: payload
  flowing ⇒ up *by construction*. A side-channel probe may only *enrich
  detail* when it connects; it never overrides a live link to report
  "down."
- Hold the verdict with **hysteresis** (§3.6) so one missed sample doesn't
  flicker the badge.

See [§15.10](delivery-rules.md#liveness-from-data).

**Generalisable rule:** read liveness/health from the primary data channel
the producer already streams on — publish status as telemetry there and
derive up/down from it; a separate out-of-band probe sits idle, gets
dropped/throttled by the edge, and false-reads "down" while the data
flows, so let it enrich detail at most, never override a live link, and
hold the verdict with hysteresis.

---

## 108. Two implementations of one host-singleton role run at once — the rewrite was deployed beside the legacy

**Symptom.** A shared value won't hold / flaps, a device or topic has two
owners, or two control stacks both actuate — right after a rewrite of a
service was installed on a host that still runs the legacy it replaces.

**Looks like.** A flaky daemon, a transport fault, "why is the setting
bouncing?" — the same stale/flapping signature as a duplicate process
(#102).

**Actually is.** The singleton guard was keyed on the **binary**, not the
**role**. A `flock` pidfile / instance check on the new daemon's own name
sees no peer, because the *other* owner is a **different implementation**
(the legacy stack, or a second daemon) of the same role. Both claim the
same device / topic / actuation authority; neither errors; the system goes
stale or flaps — and if both can actuate, that's a safety hazard, not a
cosmetic one.

**Fix.**
- Guard on the shared **resource** both implementations contend for — the
  device handle, the topic claim (§5.3), an actuation-authority lock — so a
  *foreign* implementation trips the same guard. Coexistence is fine only
  for **disjoint** resources; sharing one (a camera / encoder / bus / IPC
  ring) is a collision, not isolation.
- At **onboard**, claim the role by **disabling — not just stopping — the
  others**: "stopped" isn't "gone" (a still-*enabled* competitor re-grabs
  the resource on the next boot). Disable in **every** service scope
  (system *and* per-user), and reap orphaned holders **by exe-path /
  pattern, not just the unit** (a supervisor's children outlive the unit
  stop and keep holding the device).
- Disable the **whole set** of competitors **derived from the stack
  registry**, not a hand-kept parallel list that silently drops the next
  stack you add (§5.4).
- Apply the same claim at **two enforcement points** — a one-time
  privileged cutover (survives reboot) **and** a runtime claim on every
  activation (frees the resource now, re-asserts if a competitor crept
  back); degrade gracefully when privilege is absent.
- Diagnose a victim symptom (no device / 0 Hz with a healthy-looking
  producer) by **counting the holders** of the contended resource and
  checking whether a second stack is still *enabled*.

See [§6.8](engineering-rules.md#singleton-process).

**Generalisable rule:** a host-singleton is a *role*, not a binary — a
different implementation of the same role (a rewrite beside the legacy) is
still a duplicate; guard on the shared resource so a foreign owner trips it,
and onboard one implementation by **disabling (not just stopping) and
reaping** every other — across scopes, for the whole registry-derived set,
at both the privileged cutover and the per-activation claim — rather than
racing it.

---

## 109. The rewrite "runs" but emits different numbers than the legacy — nobody dual-ran them on real data

**Symptom.** A ported transform (a wire bridge, decoder, estimator, model
path, unit conversion) compiles, runs, and passes its unit tests — but
downstream behaviour is subtly wrong (controls hunt, perception drifts,
a value is off by a scale/sign), and nobody traces it to the port.

**Looks like.** A downstream bug in the consumer of the rewritten code.

**Actually is.** The rewrite was checked for *coverage* ("every function
crossed", #28) and for "it runs", but never for **behavioural parity** —
that it produces the **legacy's** output on real inputs. A wrong constant,
a dropped term, a rounding / endianness / off-by-one difference yields a
clean run with quietly different numbers; synthetic unit tests didn't
exercise the regime that diverges.

**Fix.**
- **Dual-run on real recorded data** (the §16.9 log of record): feed the
  same captured inputs through legacy and rewrite, assert outputs match —
  bit-identical where deterministic, within a *stated* tolerance for
  float/seeded paths.
- Make it a **cutover gate + regression guard** in CI, not a one-time
  demo; until it passes, the new path is UNVALIDATED and ships only
  gated-inert (§20.1).
- A mismatch indicts the **rewrite** (the legacy is the reference); diff
  the intermediate stages (§15.8) to find where the paths first diverge.

See [§18.8](delivery-rules.md#rewrite-parity).

**Generalisable rule:** when a rewrite's job is to reproduce an existing
transform, prove parity by running legacy and new on the *same real
recorded data* and asserting equal output (bit-identical or within a stated
tolerance) — "it runs" and "unit tests pass" don't prove it produces the
legacy's answer; gate cutover on the parity run and keep it as a regression
guard.

---

## 110. The deploy "skipped — already current," but your edit never shipped

**Symptom.** You add a version gate so onboard stops cross-building every
time ("skip if the rig already runs this revision"). It works — until an
edit you're actively testing silently doesn't reach the target: the deploy
says "already current, skipping" and ships nothing, so you debug code the
rig isn't running.

**Looks like.** A caching bug, a flaky deploy, "I definitely saved the
file."

**Actually is.** The skip-check gated on the **bare commit sha**
(`git rev-parse HEAD`), or computed "dirty" with `git diff` — which only
sees **unstaged tracked** changes. A **staged** edit or a **new untracked**
file carries the *last commit's* identity, so the gate reads "target ==
source" and skips. The version stamp wasn't sensitive to the exact change
you made, so the optimization that skips redundant builds also skips the
build that mattered.

**Fix.**
- Gate on **`sha + dirty`**, never the bare sha — an uncommitted tree must
  not match a committed target.
- Compute dirty with **`git status --porcelain`** (staged + unstaged +
  untracked), not `git diff` (§12.1). A half-blind dirty check makes the
  skip *look* safe while stranding staged work.
- A **`-dirty` stamp never matches** — always rebuild + ship it (it's
  unreproducible), or refuse it on a release path.
- Confirm "already current" against the version the **target actually
  reports** (§28.3), not a local assumption.

See [§12.3](delivery-rules.md#deploy-idempotent).

**Generalisable rule:** version-gating a rebuild/redeploy to skip
redundant work is only safe if the gating identity is `sha + dirty` with
dirty = `git status --porcelain` (catches staged + untracked) — gate on the
bare sha or a `git diff`-only dirty check and the skip hands an uncommitted
edit the committed identity, silently dropping the change you're testing.

---

## 111. One torn record at the end of a log discards the whole log

**Symptom.** A replay / upload / restore of a recorded log (or a WAL,
event log, framed stream) **fails entirely** — errors out, or yields
nothing — after the writing process was killed uncleanly (power cut, OOM,
`SIGKILL`, full disk). The data looks "corrupt," but 99% of it is intact.

**Looks like.** A corrupt/unreadable file; "the recording is lost."

**Actually is.** The writer was interrupted **mid-append**, leaving a
**torn trailing record**: the length prefix was flushed but the body was
cut short. The reader trusts the prefix and does a blind `read_exact(len)`
on the body, hits EOF, and propagates the error for the **whole stream** —
so a single unfinished tail throws away **every complete record before
it**. The last record of any crash-interrupted append log is *always*
suspect; a reader that doesn't expect it loses everything.

**Fix.**
- Treat a **mid-record EOF at the end of the stream** as a clean
  end-of-stream (`Ok(None)`), not an error — recover every complete record,
  drop only the partial tail. Reserve a hard error for a torn record with
  valid records *after* it (real interior corruption).
- Make completeness **checkable** before consuming: a length prefix or
  self-delimiting framing, ideally a per-record checksum, so "is this
  record whole?" is a positive test.
- **Test against a deliberately truncated stream** (two good records + a
  third cut mid-body → reader yields two, ends clean) — the §18.5 RED you'd
  otherwise only hit after a rig hard-powers-off in the field.

See [§16.10](delivery-rules.md#torn-record-recovery).

**Generalisable rule:** a reader of an append-only record stream must
survive a torn trailing record left by a writer killed mid-append — a
mid-record EOF at the end is a clean stop that recovers every complete
record, never a `read_exact` error that discards the whole log; the last
record is always suspect, so make completeness checkable and test against a
truncated stream.

---

## 112. Service stays DOWN after a restart — the singleton claim hit a not-yet-released port/lock and read it as "already running"

**Symptom.** Right after a restart / redeploy, the service is **down**. Its
log says *"another instance is already running"* — but `pgrep` shows none,
and the port / lock is free a moment later. The deploy's "start service"
step reported failure even though nothing was actually holding the slot.

**Looks like.** A stuck or duplicate process; "something's still holding
it."

**Actually is.** The supervisor **reaped** the old instance and started the
new one ~1 s later, but the OS hadn't **released** the singleton's claim
yet: a bound loopback **port** in `TIME_WAIT`, a `flock`/`O_EXCL` lock or
pidfile not yet freed, a device handle not yet closed. The new instance's
**one-shot** claim hit `EADDRINUSE` / lock-held and exited — a *false*
"already running." Restart outran teardown
([`Restart=always`](engineering-rules.md#restart-policy) makes that the
common path).

**Fix.**
- **Retry the claim over a bounded window** (a few seconds) on startup, so
  a transient post-reap release is waited out.
- A **genuinely-live** peer never releases → still fail after the window
  (the singleton invariant holds; you don't want two).
- For a TCP/loopback lock, `SO_REUSEADDR` softens `TIME_WAIT`, but the
  bounded retry is the general fix across lock types (flock, pidfile,
  device).

See [§6.8](engineering-rules.md#singleton-process).

**Generalisable rule:** a singleton claim is **retried with a bounded
wait** on startup — a just-reaped predecessor's release (a `TIME_WAIT`
port, a lock, a pidfile, a device handle) lags the restart, so a one-shot
claim misreads the lag as a live peer and leaves the service falsely
"already running" and down; retry briefly, then fail (a live peer never
releases).

---

## 113. The shipped bundle baked in the builder's personal config — token and all

**Symptom.** A fleet installer / app bundle handed to operators carries a
**shared bearer token** (and maybe a stale URL) — the build literally
snapshots the builder's `~/.config`. Every recipient, and anyone the
bundle leaks to, now holds the fleet secret.

**Looks like.** Convenience — "the bundle is ready to use out of the box,
no setup."

**Actually is.** A distributable artifact is a **copy machine** for
whatever's baked in. Snapshotting the builder's personal config embeds
*their* credentials into the artifact, so the secret leaks to the whole
recipient set at once and any single leak is a **fleet-wide rotation
event**. "Ready to use" smuggled a shared credential into a thing designed
to be copied widely.

**Fix.**
- Ship a **checked-in, secret-free default** (service URLs, non-secret
  targets, the SSH alias, feature toggles) — useful out of the box with
  **zero** credentials; the default lives in the repo, not snapshotted
  from a laptop.
- **Seed the credential per-device at runtime** — the operator's own SSH
  key, a provisioning step, env / keychain, a tokenless audited enrollment
  ([§26.4](delivery-rules.md#per-identity-access)) — never at bake.
- **Gate the bake**: grep the artifact and **refuse to ship** if any
  token / key / password field is present (defence in depth against a
  future careless snapshot).

See [§14.5](delivery-rules.md#secret-free-bundle).

**Generalisable rule:** never bake a secret into a shippable / extractable
artifact — ship non-secret defaults, seed the per-device credential at
runtime, and gate the ship on a no-secret grep; a baked shared token leaks
to every recipient and any leak rotates the whole fleet.

---

## 114. Fleet access by a key on every host — adding or revoking an operator means editing every host

**Symptom.** Onboarding a new operator / device means appending their
public key to **every** host's `authorized_keys`; the lists **drift**
(some hosts missed); removing someone who left requires touching every
host, so it doesn't happen and **stale access lingers** across the fleet.

**Looks like.** Per-identity keys done right (#85 / §26.4) — "we don't
share secrets, everyone has their own key."

**Actually is.** The *credential* is right, but its **placement doesn't
scale**: a key on each host is **O(identities × hosts)**. There's no
central revocation, no single enrollment, and the next host or the next
operator silently gets missed. Per-identity correctness without a
scaling story degrades into copy-paste and drift.

**Fix.**
- Stand up an **SSH user CA**; every host trusts the **CA**
  (`TrustedUserCAKeys`), not each key.
- An enrolled identity gets a **short-lived signed certificate** and
  reaches every host — enroll **once**, no per-host distribution, no
  drift.
- Revoke **fleet-wide and instantly** via a KRL; short cert lifetimes
  bound a leak.
- Guard the CA key as a **fleet-wide secret** (hardened host, audited
  signing, scheduled rotation); harden the now-uniform front door
  (rate-limit / fail2ban).

See [§26.5](delivery-rules.md#fleet-access-ca).

**Generalisable rule:** at fleet scale, replace `authorized_keys`-per-host
with a central SSH user CA issuing short-lived per-identity certs — hosts
trust the CA, an identity enrolls once and reaches all, certs expire on
their own, and a KRL revokes one identity fleet-wide; per-host key lists
are O(identities × hosts), drift, and have no central revocation.

---

## 115. The version tag points at the wrong bytes — moved or poisoned, consumers silently get stale code

**Symptom.** Consumers pinned to `vX.Y.Z` of a shared lib either **don't
pick up your fix** (the resolver says nothing changed / the tag is
"unusable") or **build against wrong/stale content** — even though "you
published `vX.Y.Z`." Re-pushing the same tag doesn't fix it.

**Looks like.** A caching glitch, a flaky registry, "cargo/go/pip is
broken."

**Actually is.** The tag broke its contract that *number → exact bytes*:
- **Moved** — re-pointed to new content, but consumers' resolvers cache
  `tag → sha` (lockfile, registry, CI mirror), so the move doesn't
  propagate; `git push` even refuses it without `--force`.
- **Poisoned** — a broken / partial publish tagged the wrong commit, or
  the tag push landed while the content push didn't (the #91 "a dropped
  write isn't a failed write" family), so the tag exists but carries the
  wrong bytes.

Either way the version number is now lying, and every consumer that cached
it is stuck on bad content.

**Fix.**
- Treat published tags as **immutable** — never move/overwrite one a
  consumer pins.
- **Burn the bad number and cut a fresh higher one** (poisoned `v0.1.12` →
  re-cut `v0.1.13`, never re-push `v0.1.12`), so cached-bad consumers are
  forced onto a clean tag. A burned number is dead forever.
- After publishing, **verify from a clean fetch** that the tag carries the
  intended version stamp (§28.3) + content before announcing — a local
  "publish succeeded" ≠ the remote tag's bytes (#91).

See [§28.5](engineering-rules.md#immutable-version-tag).

**Generalisable rule:** a published version tag is immutable — never move
it (resolvers cache `tag → sha`, so a moved tag breaks or no-ops), burn a
mis-cut number and cut a fresh higher one rather than re-pointing it, and
verify the tag's content from a clean fetch before consumers depend on it.

---

## 116. The teardown path crashes the worker it was supposed to stop

**Symptom.** Stopping / closing / switching away from something segfaults
or corrupts state **on exit** — a video decoder, a subprocess, a worker
thread. It "works" in steady state; the crash is at shutdown. Reads as
"flaky teardown."

**Looks like.** A bug in the worker, a race, a bad library.

**Actually is.** The teardown **immediately** sent `SIGTERM`/`SIGKILL`
(or dropped the handle so the runtime aborted the task) while the worker
was **mid-operation** — holding a native/codec context, a `dlopen`'d
library's state, a lock, a half-written buffer, a DMA in flight. The
forced kill raced the operation; the resource was never released cleanly.
The teardown path *itself* is the fault — the worker would have exited
fine a few milliseconds later.

**Fix.** Make teardown **two phases**:
- **Cooperative stop first**: set a `stop_event` / cancellation flag the
  worker's loop checks each iteration (or close its input), and **wait,
  bounded**, for it to reach a safe point and exit on its own — releasing
  its codec / handle / lock.
- **Forceful escalation only on a hang**: `SIGTERM` → `SIGKILL` *after*
  the bound expires, as the fallback for a stuck worker, never the first
  move.

See [§6.9](engineering-rules.md#cooperative-stop).

**Generalisable rule:** stop a worker (thread / subprocess / task)
cooperatively before forcing it — signal a stop flag and wait a bounded
time for it to exit on its own (releasing its native resource), escalating
to `SIGTERM`/`SIGKILL` only if it hangs; an immediate kill mid-operation
crashes or corrupts, making the teardown path itself the bug.

---

## 117. A benign non-zero aborts the whole build / onboard — an empty glob or an absent optional under `set -e`

**Symptom.** A build or onboard dies on something that isn't a failure: a
wheel-freshness `ls -t build/*.whl` on a clean tree, a `grep` with no
hits, an `install some_optional.py` for a file an older checkout legitimately
lacks ("cannot stat …"). The *real* work would have succeeded; the script
aborts before it.

**Looks like.** A real build break; "the bundle is corrupt"; a flaky onboard.

**Actually is.** `set -e` / `pipefail` — correctly used for the essential
steps (§18.4) — also kills the script on a **benign** non-zero: a glob
that matches nothing, a probe with no result, a *genuinely optional* input
that's absent. The step that exited non-zero wasn't the work; it was
auxiliary to it, and its "nothing to do" was misread as "the build
failed."

**Fix.**
- Guard the **genuinely-optional / may-be-empty** step so it degrades and
  logs: `… || true`, an `[ -f file ]` existence check before consuming it,
  `[ -n "$(glob)" ]`, consume-when-present.
- Ship a script and the files it references as **one bundle**; an optional
  sibling goes in an *optional* file-set (shipped when present, absence is
  not an error), and the consumer reads it with `if [ -f ]` (mirrors a
  guarded import, §6.6 / paired-deploy §13.4).
- **Don't** answer this with a blanket `|| true` on the **essential**
  work — that's the §18.4 false-green. The discriminator is "is this step
  the work, or auxiliary to it?"

See [§18.4](delivery-rules.md#propagate-exit-code).

**Generalisable rule:** under `set -e`/`pipefail`, a benign non-zero from
an auxiliary or optional step (an empty glob, an absent optional file, a
best-effort probe) must degrade — guard it (`|| true` / existence check)
so it doesn't abort the build/onboard, while the essential work still
fails loud (never blanket-swallow that).

---

## 118. Untrusted content became code — a job name ran as `<script>`, a field as SQL

**Symptom.** A value a user controls — a job/booking name, a filename, a
device-reported string, a row another user wrote — ends up **executing**:
a stored name like `<script>…</script>` runs in every viewer's browser
(XSS), an input like `'; DROP TABLE …` runs in the DB, a filename with
`; rm -rf` runs in a shell.

**Looks like.** A weird display glitch, a "corrupt" record, a flaky query
— or nothing, until it's exploited.

**Actually is.** Untrusted data was **interpolated raw into a string an
interpreter then parses** (HTML/JS, SQL, a shell, a template). The
interpreter can't separate your structure from the attacker's payload, so
the data becomes syntax. Hiding the field in the UI or escaping it on the
client doesn't help — the sink is on the server.

**Fix.**
- **Output**: render through an **auto-escaping** templating engine, and
  encode for the *exact* context (HTML body vs attribute vs JS vs URL).
  Encode where you emit, not where you store.
- **Input**: **parameterize, never interpolate** — bound query params
  (prepared statements), an **argv array** for subprocesses (never
  `shell=True` / a concatenated command), a real parser for the format.
  Allowlist/length-check on top as defence in depth.
- Treat **stored** values as untrusted when you render them (stored XSS);
  the danger is at the sink, so every sink encodes for itself.

See [§26.6](delivery-rules.md#encode-untrusted).

**Generalisable rule:** untrusted data crossing into an interpreter
(HTML/JS, SQL, shell, template) must be encoded for that sink — auto-escape
on output in the right context, parameterize (bound params / argv array) on
input — never interpolated raw, or the data becomes code (XSS, SQL/command
injection).

---

## 119. The sensor reads "invalid" — the daemon lacks the OS group to open its device

**Symptom.** A device daemon publishes nothing (0 frames, or it crashed at
startup); downstream alerts "sensor data invalid / no GPS / no signal."
The hardware is fine — opening the device **by hand in a shell works**.

**Looks like.** Dead hardware, a bad cable, a driver bug, a flaky sensor.

**Actually is.** The daemon, running under the supervisor as a user, lacks
the OS **group / permission** to open the device node — a serial TTY is
`root:dialout` mode 660. You have the group **interactively**; the
**service** doesn't — its groups are the unit's (`User=` /
`SupplementaryGroups=`), not your shell's. The open returned
`PermissionError [Errno 13]`, and the daemon **swallowed it** (slept
forever) or **crashed once and wasn't restarted** — so the failure
surfaced downstream as "no data," pointing you at the hardware.

**Fix.**
- Grant the access **in the unit** (`SupplementaryGroups=dialout`, a
  capability, or a udev ACL), version-controlled and deployed with the
  service. Verify against the *service's* environment, not your shell.
- Make a device-open failure **loud**: retry + WARNING (§18.9) and/or an
  operator "can't reach device" indicator (§3.5); never silent-sleep or
  one-shot-crash-stranded (`Restart=always`, §6.2).
- The OS error may be **wrapped** by a library (pyserial →
  `SerialException`); match the cause / errno, not the narrow type.

See [§6.10](engineering-rules.md#device-access-in-unit).

**Generalisable rule:** a daemon's OS access to its devices is provisioned
in the service unit (group / capability / udev ACL) — the service runs with
the unit's permissions, not your interactive shell's — and a device-open
permission failure surfaces loudly, never silently as "no data"; otherwise
you debug the hardware instead of the missing group.

---

## 120. `Restart=always` but the service is permanently DOWN — the start-limit gave up

**Symptom.** An "always-on" service (`Restart=always`) is dead for
minutes; a consumer shows **frozen / stale** data ("carState delayed", "no
updates", "video not streaming"). Restarting it by hand fixes it — but it
shouldn't have stayed dead.

**Looks like.** A bug in the service, a transport stall, a flaky
dependency — so you go hunting downstream.

**Actually is.** The unit flapped fast enough — a dependency-flap window,
repeated operator relaunches, a crash-loop — to hit systemd's **start-limit**
(default `StartLimitBurst=5` / `StartLimitIntervalSec=10s`): systemd
**gave up** and left it `failed` *indefinitely*. `Restart=always` is set,
but the start-limit overrides it. Often *triggered* by a too-aggressive
`TimeoutStopSec`: a slow stop (a child hung in a codec/DMA wait) overruns
the timeout → `SIGKILL → failed(timeout)`, and each such failure burns the
restart budget.

**Fix.**
- For a must-keep-trying service, **`StartLimitIntervalSec=0`** to disable
  the give-up guard — in the **`[Unit]`** section (silently no-op'd if put
  in `[Service]`).
- Keep **`TimeoutStopSec` generous** so a slow stop doesn't escalate to
  `failed` and spend restart budget.
- **First diagnostic** for "stale/frozen data" from a supervised service:
  `systemctl is-active <unit>` — a `failed`/`inactive` unit is the whole
  story; stale `/dev/shm` values + a rate probe reading the dead producer's
  last values mislead you elsewhere (§15.10).

See [§6.2](engineering-rules.md#restart-policy).

**Generalisable rule:** `Restart=always` is necessary but not sufficient —
systemd's default start-limit wedges a fast-flapping unit in `failed`
forever; a must-always-run service disables the limiter
(`StartLimitIntervalSec=0` in `[Unit]`) and keeps `TimeoutStopSec` generous
so a slow stop doesn't escalate and burn the budget, and you diagnose a
"stale data" report with `systemctl is-active` first.

---

## 121. A real fault shows as benign — the indicator's "off" colour was a fixed default, not per-site

**Symptom.** An operator misses a real fault because its indicator looks
*normal*: a down link, a dropped control authority, or a stale feed shows
the same neutral gray as a feature that's simply switched off. (Or the
inverse — a normally-absent signal glows alarming red and operators learn
to ignore it.)

**Looks like.** The monitor "works" — it *does* change colour, and the
colours come from the shared palette, so nothing looks wrong in review.

**Actually is.** The boolean→colour helper used a **fixed "off" colour**,
but *off / false / absent* is **per-site semantic**: normal at one
indicator (gray), a **fault** at another (red), merely noteworthy at a
third (amber). The same `False` rendered uniformly is right at some sites
and **dangerously wrong** at others — a link that should be live, painted
"off = gray", reads as fine.

**Fix.**
- The status→colour helper takes the off-meaning **per call site**
  (`off=IDLE|WARN|BAD`); choose it deliberately at each indicator.
- "Off is normal" → IDLE/gray; "off is a problem" (a link / authority that
  should be live) → BAD/red; "off is noteworthy" → WARN/amber.
- Colours still come from the **one shared palette** (§8.25); only the
  off-*semantic* is per-site. Carry state in shape too (filled vs hollow
  dot), not colour alone.

See [§8.25](ui-rules.md#theme-colors).

**Generalisable rule:** a status indicator's off / false / absent state is
per-site semantic — a shared status→colour helper takes the off-meaning at
each call site (`off=IDLE|WARN|BAD`), because the same `False` is normal at
one indicator and a fault at another; a fixed off-colour paints a real
fault benign (the operator misses it) or a normal absence alarming (cries
wolf).

---

## 122. The submodule broke the build — empty dir, or "transport 'file' not allowed"

**Symptom.** Consumer builds / clones fail or silently get **empty files**
where a shared submodule (rules, config, a `.proto`) should be:
`fatal: transport 'file' not allowed`, or the directory is empty, or CI
can't reach the submodule URL.

**Looks like.** A flaky network, a broken dev setup, "did you forget
`--recurse-submodules`?"

**Actually is.** A git **submodule** was used for content that must merely
**be present** — so it carries a fetch / init / transport step that fails
independently of the content. git 2.38+ **blocks the `file://` transport**
(CVE-2022-39253) for submodules and `url.…insteadOf` local-mirror rewrites
→ `transport 'file' not allowed` on every build that recurses; a clone
without `--recurse-submodules` leaves empty dirs; a private/unreachable URL
fails in CI. The content has no build or version lifecycle, so the
submodule machinery buys nothing and only adds failure modes.

**Fix.**
- **Vendor committed copies** of the files into each consumer; propagate
  with a **path-limited copy-in-and-commit** step (WIP untouched). Fresh
  clones get them with zero setup — nothing to fetch or init.
- Reserve submodules / prefix-deps for **versioned code** with its own
  lifecycle (§13.2); even then the URL must be reachable from every
  consumer (never `file://`) and consumers must init it.

See [§13.5](delivery-rules.md#vendor-not-submodule).

**Generalisable rule:** content that must merely be present in every repo
(rules, config, schemas, templates) is vendored as committed copies with a
path-limited copy-in step — not a git submodule, which breaks builds on a
missing `--recurse-submodules`, a `file://` / local-mirror URL (blocked by
git 2.38+, CVE-2022-39253), or an unreachable URL; reserve submodules for
versioned code (§13.2).

---

## 123. A gray or smeared video tile at the correct size — the encoder is starved, not the GUI

**Symptom.** A teleop / operator GUI shows flat-gray or vertically-streaked
camera tiles (a faint real scene visible under them, overlays drawing fine
on top), each labelled the **correct** resolution, even on **fresh (0 ms)**
frames. Or one tile hangs at "waiting for stream" while a sibling streams
cleanly.

**Looks like.** A GUI render bug — a stride / reshape mismatch, a wrong
decode size, a resolution mismatch, a broken frontend.

**Actually is.** An **encoder-side** fault on a thin uplink — and the
geometry being *right* is the tell that it is **not** the renderer (a
display that reshapes on the writer's own `w,h` can't emit a
correctly-sized wrong picture). Two flavours:
- **Gray / streaked at the correct size = bitrate starvation.** The
  rig→operator uplink is thin, the ABR floors the bitrate (e.g. 77 kbps);
  at that size each frame is ~200 bytes = DC-only macroblocks = flat-gray
  blocks (the "streaks" are surviving low-frequency luma of real walls).
  The SPS carries the true size inline every IDR — which is *why* the tile
  shows the correct res; sending resolution "dynamically" fixes nothing,
  the size was already right.
- **Stuck "waiting for stream" = IDR starvation.** The stream lost its
  reference frame and the keyframe-on-loss recovery is a silent no-op —
  the force-IDR request was wired to the wrong platform API (a GObject
  *property* the encoder exposes only as an action *signal*), so it forces
  zero IDRs and the decoder never re-seeds.

**Fix.**
- **Rule the GUI out by construction first** — confirm the display reshapes
  with the writer's own `w,h` guarded on `len == w*h*3` and the res is
  32-aligned (no YUV stride padding). If so, the renderer is not the bug.
- **Decode the encode bytes at the source** (diagnostic-first): dump the
  `EncodeData` topic's `data` to a `.h264` and decode with **plain ffmpeg**
  on the rig (standard Annex-B, inline SPS/PPS). Gray → the *encode* is
  starved (check the bitrate; ~77 kbps = thin uplink). Grab a raw capture
  frame too, to confirm capture is clean.
- **Fix the cause, not the symptom** — shrink the encode resolution by a
  **link-profile preset applied via a camerad restart** (never a live
  re-cap, #65) so a thin link carries fewer bytes/pixel; and wire force-IDR
  to the platform's real API, verifying one IDR/encoder with no warning.

See [§3.4](engineering-rules.md#control-feed-freshness).

**Generalisable rule:** a corrupted video tile at the *correct* geometry is
a producer/encoder-side fault (starved bits or a missing keyframe), never a
renderer bug — rule the GUI out by construction, then decode the encode
bytes at the source; gray-but-sized = the bitrate floor (shrink res by a
preset via restart), stuck "waiting for stream" = a missing IDR (fix the
keyframe request to the platform's real API).

---

## 124. Bulk upload between two co-located hosts 502'd — it detoured over the thin WAN edge

**Symptom.** A fleet upload / "pull latest" arms but no data lands; the
uploader logs `upload_start` → `upload_failed 502` on **every** multi-MB
file, while a 16-byte probe PUT returns `201`.

**Looks like.** A broken uploader, a dead endpoint, a server-side 502, an
auth problem.

**Actually is.** The transfer path is **bandwidth-starved, and both
endpoints were on the same LAN the whole time.** The rig (`192.168.10.220`)
and the datacenter Server (`192.168.10.252`) share a LAN, yet the upload
detours rig → thin metered **WAN** up (~90 KB/s) → cloud edge → tunnel back
down → Server. A 6 MB PUT takes ~70 s at 0.09 MB/s — right at the read
timeout; when teleop competes for the uplink it tips past the timeout → the
edge **502s** → the force-upload flag never clears → the backlog never
drains. Small files pass, big ones fail — the signature of **bandwidth-bound
(not parallelism-bound)** starvation ([§9.9](engineering-rules.md#shared-resource-contention)).

**Fix.**
- **Route bulk data over the LAN when both endpoints are co-located.**
  Resolve the peer's on-LAN address from the registry heartbeat (`lan_ip`,
  [§23.4](delivery-rules.md#reach-a-fleet-host-via-its-registry)); if it's
  on your LAN, PUT straight to it at LAN speed, bypassing the WAN edge
  entirely. Fall back to the cloud relay **only** when off-LAN, and derive
  the target from the registry — never a new env var ([§7.5](engineering-rules.md#env-var-config)).
- **Size the timeout to `bytes ÷ worst-case bandwidth`, not a constant**
  ([§9.9](engineering-rules.md#shared-resource-contention)), so a
  legitimately slow transfer isn't cut mid-flight.

See [§23.4](delivery-rules.md#reach-a-fleet-host-via-its-registry) ·
[§9.9](engineering-rules.md#shared-resource-contention).

**Generalisable rule:** bulk data between two co-located fleet hosts takes
the **LAN path**, not the metered WAN edge — resolve the peer's `lan_ip`
from the registry and transfer LAN-direct, relaying through the cloud only
when off-LAN; a big-file-only failure (small probes pass) on a thin uplink
is bandwidth starvation, not a broken endpoint.

---

## 125. A renamed heartbeat field silently greyed out every consumer still reading the old name

**Symptom.** A per-host feature (an "upload" button, a "pull latest"
action) is greyed / disabled or reads "no net" for a host that is plainly
connected and streaming — and only *some* UIs are affected.

**Looks like.** The host is offline, the network is down, a flaky
heartbeat, a per-UI bug.

**Actually is.** The producer **renamed a field in a schemaless
telemetry / heartbeat payload** (`network_type` → `info.uplink_kind`) and
the rename swept only one repo's consumer. Every other consumer still does
`info.get("network_type")` and gets `""` — no crash, no type error, just a
**falsy read that fails the gate**. Because the payload is an untyped dict,
nothing flagged the drift at compile time; it's a
[§18.2](delivery-rules.md#migration-completeness) migration-incompleteness
across *repo* consumers, surfacing as a silent empty string.

**Fix.**
- **Sweep every consumer of the renamed field across all repos** — a
  schemaless dict has no compiler to find them; grep the old key
  fleet-wide.
- **Read with an old-name fallback through the transition**
  (`uplink_kind or network_type or ""`) so an un-bumped consumer degrades
  instead of blanking.
- **Treat the heartbeat / telemetry payload as an append-only schema**
  ([§4](engineering-rules.md#schemas)) — renaming a key is a breaking
  change even when it's "just a dict."

See [§18.2](delivery-rules.md#migration-completeness) ·
[§4](engineering-rules.md#schemas).

**Generalisable rule:** renaming a field in a schemaless telemetry /
heartbeat payload is a breaking, **silent** migration — the untyped read
returns `""` not an error, so every un-updated consumer quietly fails its
gate; sweep all consumers across repos, read with an old-name fallback
through the transition, and treat the payload as append-only.

---

## 126. A fixed bug came back after an unrelated change — the fix had no permanent guard

**Symptom.** A defect you *know* you fixed weeks ago is back, and the commit
that reintroduced it was "fixing something else entirely" — or was a mechanical
cleanup (`ruff --fix`, remove-unused-import, a formatter, a refactor). It feels
like whack-a-mole: fix B, bug A returns; fix A, later bug C brings it back.

**Looks like.** A hard/flaky bug, a mysterious regression, bad luck, "the code
is fragile."

**Actually is.** **The original fix left no executable guard**, so nothing red-
flagged its reversal. Two common mechanisms: (a) the fix was *deliberate code
that looks like a mistake* — a re-exported symbol nothing local uses, a specific
list ORDER, a magic constant, a redundant-seeming check — and an auto-fixer or
"cleanup" removed it (seen this very session: `ruff --fix` stripped a re-export
a test relied on); or (b) two fixes are a **constraint pair** with opposing
requirements on the same code, and satisfying the new one silently violated the
old one because the old one was never encoded as a test.

**Fix.**
- **Ship a permanent regression guard with every fix**
  ([§18.5.1](delivery-rules.md#regression-guard)) — a test that goes RED if the
  bug returns and stays in the suite forever. No guard = an unenforced fix.
- **When a later change reds an old guard, reconcile BOTH requirements** — never
  delete/loosen/`xfail` the guard to get green. That red bar is the system
  catching you trade one fix for another.
- **Mark deliberate-looking-wrong code in place** — `# noqa: <rule>` / `// keep:`
  with the reason + a pointer to its guard, so no tool or refactor reverts it.

See [§18.5.1](delivery-rules.md#regression-guard) ·
[§18.5](delivery-rules.md#test-first).

**Generalisable rule:** a bug that keeps coming back is not hard, it is
**unguarded** — a fix isn't done until a permanent test reds on its
reintroduction, you reconcile (never silence) that test when a later change
fires it, and any fix that looks like a mistake is marked with its reason +
guard so no cleanup can quietly undo it.

---

## 127. A code fix "didn't take" no matter how many times you edited — the running process was a COMPILED copy, not the source you touched

**Symptom.** You edit a file, deploy/restart, test — no change. You edit again,
add logging, restart — still the old behaviour, and your new log line never
prints. It feels like the change is being ignored or cached; you start doubting
the fix itself.

**Looks like.** A stale cache, a deploy that didn't land, a wrong config, a
flaky bug, "the fix doesn't work."

**Actually is.** **The process running is not the file you edited.** The
runtime loads a *different* artifact than the source you're changing —
classically a **compiled build** (Cython `.so`, a bundled/frozen binary, a
`.pyc`, a container image layer, an installed wheel in `site-packages`) while
you edit the **source tree** next to it. Seen this session: a rig ran the Cython
`modeld.*.so` from `~/.local/lib/…/site-packages`, but the edits + all
diagnostics went to a `~/unofsd-build/…/modeld.py` build-cache next to it; a gdb
`py-bt` showing `modeld.c` (not `.py`) is what finally revealed it — after three
"failed" fixes that were never executed.

**Fix.**
- **Before iterating, confirm which artifact actually runs.** Ask the live
  process, don't assume: `readlink /proc/<pid>/exe`, `python -c "import m;
  print(m.__file__)"` *in the runtime's env* (not yours), `gdb -p <pid> -batch
  -ex py-bt`, `lsof -p <pid> | grep <module>`. A `.so`/`.c` frame or a
  `site-packages` path means edits to the source tree do nothing.
- **Edit the source of record + rebuild/redeploy the artifact** — never
  hot-patch the copy that isn't loaded; a compiled runtime needs the build step.
- **Make source-vs-runtime unmistakable at the location** — a build cache /
  staging tree next to a compiled deployment gets a marker (`README.BUILD-CACHE`)
  naming where the runtime lives, so nobody edits the wrong copy.

See [§18.5.1](delivery-rules.md#regression-guard) ·
[§18.9](delivery-rules.md#inject-the-fault).

**Generalisable rule:** when a change "has no effect", first prove the process
is actually running the file you edited (ask `/proc`, the importer, or a live
backtrace) — a compiled `.so` / frozen binary / installed wheel / container
layer runs INSTEAD of the adjacent source, so hot-patching the source tree is a
no-op; edit the source of record, rebuild, and mark any build-cache tree so the
wrong copy can't be mistaken for the runtime.

---

## 128. A validator existed and was correct — but one write path didn't call it

**Symptom.** Input validation is present, tested, and demonstrably works when
you call it directly. Bad data still lands in storage. The validator's own unit
tests are all green, so suspicion falls on the parser, the client, or the
storage layer.

**Looks like.** A missing validation rule, a parser bug, a client sending
something new, "we need stricter checks."

**Actually is.** **The check is fine; a write path skips it.** The same value
reaches storage through more than one code path (manual submit, re-recognize,
batch import, a retry/repair job), and the guard sits on the path someone was
looking at when they wrote it — not at the point where the value is *written*.
Seen in the wild: a money parser rejected NaN/inf correctly, but one assignment
built the amount inline with a bare `float()` and never went through it, so
`"nan"` poisoned every `sum()` over that column; separately, four upload
endpoints each read their own multipart part, so a format whitelist added to one
would have left three open.

**Fix.**
- **Put the guard at the choke point, not the entry point.** Validate where the
  value is *written* or in a single shared reader all entries must pass through
  (one `_read_receipt_part` that every upload route calls), so a new fifth route
  inherits it instead of forgetting it.
- **Enumerate the write sites before trusting a validator.** Grep for the
  destination (the field/column/table), not the validator name — the paths that
  *don't* mention it are the ones at risk.
- **When a guard can't be centralised, make the bypass loud.** Log a warning on
  the unvalidated branch rather than silently accepting; silence is what let
  this survive.

**Generalisable rule:** a correct validator proves nothing about coverage — the
question is never "is the check right?" but "does every path that writes this
value go through it?", so anchor the check at the single point of write (or one
shared reader) and enumerate write sites by grepping the destination, because
the path that skips validation is invisible in the validator's own tests.

---

## How to add to this list

If you debugged something for >30 minutes and the symptom didn't
match the root cause, that's a candidate. Open a PR with: symptom
→ looks like → actually is → fix. Phrase it generically so the
*shape* of the bug is what's remembered, not the specific service.

Three lines per entry beats ten. Fast recognition is the point.
