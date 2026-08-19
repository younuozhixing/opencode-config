# UI architecture rules

Companion to [`engineering-rules.md`](engineering-rules.md). The UI
section (**§8**) lives here: at 28 sub-rules it is the largest single
body of rules and navigates better on its own. **Nothing was
renumbered** — the section is still §8, the sub-rules still §8.1 … §8.30,
and every slug anchor is unchanged, so existing cross-references keep
resolving; only the file moved. References to *other* sections point at
[`engineering-rules.md`](engineering-rules.md)`#…`.

Same conventions as the main file: stable numbers, slug anchors, an
observable failure mode per rule. Jump by task via the trigger index in
[`CLAUDE.md`](CLAUDE.md); scan every rule in one line via
[`quickref.md`](quickref.md).

---

<a id="ui-architecture"></a>
## 8. UI architecture

> **When this fires** — any interactive-surface change.
>
> **Do** — read **[`ui-rules.md`](ui-rules.md)** — §8 and all its sub-rules live there (snapshot pattern, wire-state mirror, masking, the house design language).

This is the largest section — 28 sub-rules. They group into six
themes; read the one your change touches, not all of them. (Same
stable-number, regroup-only convention as the top-level parts:
sub-rule numbers never move, the groups are a reading map.)

- **8.a Structure & data flow** — how the UI is wired to its data.
  - [8.1 Layer boundary](#ui-architecture) ·
    [8.2 Snapshot pattern (house style)](#snapshot-pattern) ·
    [8.3 Lifecycle: one hook, LIFO](#lifecycle-hook) ·
    [8.10 Target / device hot-swap](#device-hot-swap) ·
    [8.11 Orphan sweep on startup](#orphan-sweep) ·
    [8.12 Cross-cutting primitive helpers](#orphan-sweep) ·
    [8.30 Log to Rerun for telemetry/debug viz](#rerun-visualization)
- **8.b Performance under load** — keep the frame loop cheap.
  - [8.5 Per-frame snapshot cache](#per-frame-cache) ·
    [8.6 Coalesced publishes](#coalesced-publishes) ·
    [8.7 Virtualization for unbounded views](#virtualization) ·
    [8.8 GPU texture lifecycle](#texture-lifecycle) ·
    [8.16 Frame-rate-independent animation](#animation)
- **8.c Robustness** — one bad view / dead element can't take the UI down.
  - [8.4 View-level error containment](#view-error-containment)
    (per-stream isolation + capped recovery live in
    [§10](engineering-rules.md#per-stream-isolation))
- **8.d Text & theming** — render correctly in every locale and theme.
  - [8.14 Fonts & the glyph invariant](#fonts) ·
    [8.15 Multilingual support (i18n)](#i18n) ·
    [8.24 Disable framework default chrome](#framework-chrome) ·
    [8.25 Colours from the active theme](#theme-colors) ·
    [8.28 The house visual language (Song-minimalist)](#house-design-language)
- **8.e Operator truth & safety** — the screen reflects reality, leaks nothing.
  - [8.9 Input arbitration (safety)](#input-arbitration) ·
    [8.19 Mirror the wire, not local input (safety)](#wire-state-mirror) ·
    [8.20 Overlays: "silent" vs "broken"](#overlay-staleness) ·
    [8.21 Live fault-localization](#fault-localization) ·
    [8.22 Mask infra detail on always-visible surfaces](#masking) ·
    [8.27 Confirm a control action by read-back, not write-OK (safety)](#verify-after-write) ·
    [8.29 Re-runnable operator action (one-click recovery)](#rerunnable-action)
- **8.f Layout & device fit**
  - [8.18 The status banner pattern](#banner-pattern) ·
    [8.23 Size for the deployment context](#device-sizing)
- **8.g Legacy** — [8.26 Retained-mode (Qt) addendum](#qt-legacy)

<a id="layer-boundary"></a>
### 8.1 Layer boundary

A `services/` or `app/` layer that publishes state to the UI lives
below the widget / window tree. **No framework-level widget /
window imports below this boundary.** If you need to import a
widget class, the code belongs above the boundary; if you need to
publish state without touching widgets, it belongs below.

This rule applies to both immediate-mode and retained-mode UIs.
Crossing the boundary is the start of every "the service knows
about the dialog" bug we've shipped.

<a id="snapshot-pattern"></a>
### 8.2 House style: the snapshot pattern (immediate-mode UIs)

New UIs are built around an **immediate-mode render thread plus
service-published snapshots**. Every service exposes an immutable
snapshot dataclass; the render thread reads the latest snapshot
per frame. There is no callback / signal / slot in the hot path.

```python
@dataclass(frozen=True)
class FooSnap(FrozenSnapshot):
    value: float = 0.0
    live: bool = False
    last_update_mono: float = 0.0

class FooService(SnapshotSource[FooSnap]):
    def __init__(self, transport):
        super().__init__(FooSnap())
        self._t = threading.Thread(target=self._subscribe, daemon=True)
        self._t.start()
    def _subscribe(self):
        for msg in self._transport.subscribe("foo"):
            self._publish(FooSnap.from_msg(msg))
    def stop(self):
        self._transport.unsubscribe("foo")
```

The render thread does:

```python
snap = app_state.foo.snapshot()
draw_panel(snap)
```

**Two invariants the producer must hold:**

1. The current snapshot is always a **fully-valid value** — no
   half-published state. Build the new snapshot in full, then swap
   it under a single lock.
2. Snapshots are **immutable**. Once a consumer holds a reference,
   it can iterate, pass it around, even hold it for a frame
   without the producer mutating fields under it.

Together these mean the UI can read at any time without locks
(`snapshot_unlocked()` for hot paths), without coordination, and
without ever observing a torn write. Worker dies? UI keeps reading
the last published snapshot — stale, but not crashing.

**The `live` convention.** Every subscriber-style snapshot carries
a `live: bool`. The worker sets `live=True` when a message
arrives, `False` after a liveness timeout of silence. The UI
greys out / changes badge colour when `live` is false — operators
instantly see "subscriber down" without us instrumenting every
link.

<a id="lifecycle-hook"></a>
### 8.3 Lifecycle: one hook, LIFO at exit

Cleanup goes through a single registry, not per-dialog teardown
code. Each worker / subprocess / background thread registers a
`stop` callable once at construction:

```python
gamepad = GamepadThread(); gamepad.start()
lifecycle.register("gamepad.stop", gamepad.stop)
```

On exit (signal handler or main-loop return), the registry runs
every hook in **LIFO order**. Dependencies point backward in time
— the gamepad subprocess must stop before the joystick service
that consumes it, which must stop before the transport it
publishes through — and LIFO gives this for free.

**Signal handlers route through the same shutdown path** then
`os._exit(0)`. Don't route through the UI framework's `quit()` —
native event loops on macOS don't reliably deliver Python signals
to bytecode boundaries.

<a id="view-error-containment"></a>
### 8.4 View-level error containment

Immediate-mode frameworks typically treat **any exception
escaping a per-frame callback as fatal** — the whole UI crashes.
With many dialogs, the cost of a single bug in any one of them is
the whole app down.

Wrap each top-level draw function in a `@safe_view`-style
decorator that:

1. Logs the stack trace (rate-limited to once per N seconds per
   view, so a per-frame exception doesn't flood the log).
2. Renders a one-line error banner where the view would have
   drawn.
3. **Unwinds the framework's begin/end stack back to the depth the
   view started at** — a `begin` / `begin_child` / `push_*` the
   crashed view left open otherwise aborts the frame
   (`Missing EndChild()`). Prefer the framework's **own
   error-recovery** over hand-balancing: snapshot its stack-size /
   error-recovery state *before* the view, and on exception call
   its recover-to-snapshot routine (e.g. Dear ImGui's
   `error_recovery_try_to_recover_state`) to emit the matching
   `End` / `EndChild` / `Pop*`. That unwinds *every* open scope —
   including ones the wrapper can't see, which is the gap a
   per-call-site `try/finally` can't close. Disable the framework's
   **assert-on-recovery** (e.g. `config_error_recovery_enable_assert
   = false`) so the recovery path doesn't itself abort. (See
   anti-pattern #13.)
4. Returns — the frame loop keeps ticking.

The cost is one Python try/except per call (≈300 ns); the benefit
is that any new dialog bug is degraded from "GUI down" to "this
one widget shows a banner."

<a id="per-frame-cache"></a>
### 8.5 Per-frame snapshot caching

Multiple panels routinely read the same service in one frame.
Each `snapshot()` takes the service's lock; for a dozen services
× 60 fps × multiple readers that's thousands of lock acquires per
second.

Provide a per-frame cache:

```python
def draw_panel(svc):
    snap = frame_snapshot(svc)   # same snap across views this frame
    draw(snap)
```

Invalidation is automatic — keyed off the framework's frame
counter. The producer keeps publishing; the cache simply
serialises all readers within a single frame to one lock acquire.

<a id="coalesced-publishes"></a>
### 8.6 Coalesced publishes for streaming output

Streaming-log dialogs and similar high-rate producers must **not**
publish per event. Every line that writes `lines=tuple(buffer)`
into a snapshot is O(N) per line; at 30 lines/s on a 2000-line
buffer that's 60k element copies/s plus a snapshot lock per line.

Publish at the smaller of **N events** or **T milliseconds**.
A `collections.deque(maxlen=N)` absorbs intermediate events; only
the boundary snapshot reflects them:

```python
_PUBLISH_BATCH_N = 10
_PUBLISH_INTERVAL_S = 0.05   # 50 ms — well below operator perception
```

50 ms is below operator perception threshold (~100 ms), so the
UI still feels real-time.

<a id="virtualization"></a>
### 8.7 Virtualization for unbounded views

Any view backed by an unbounded buffer (log lines, table rows,
search results) must virtualize — only the visible viewport is
parsed and drawn. ImGui has `list_clipper`; retained-mode
frameworks have model/view + viewport. A 4000-line buffer renders
the same ~40 visible lines per frame regardless of buffer size.

Auto-scroll for streaming content: capture the scroll position
**before** drawing this frame's lines. If the user was at the
bottom, pin to the new bottom; if they scrolled up, leave the
scroll position alone (auto-scroll fighting the operator is the
bug to avoid).

<a id="texture-lifecycle"></a>
### 8.8 GPU texture lifecycle

Every GPU texture in the UI follows one of two patterns:

**A. Library-managed cache** (e.g. an `image_display` helper that
keys by label). Mark the texture as dirty only when the underlying
buffer's sequence number advanced — otherwise the library
re-uploads the same pixels every frame:

```python
last_seq = state.last_seen_seq.get(key, -1)
params.refresh_image = (seq != last_seq)
state.last_seen_seq[key] = seq
```

Per-frame redundant upload is invisible until you measure it
(then it's tens of MB/s of needless GPU bandwidth).

**B. Owning handle in an LRU cache** (e.g. for hundreds of small
tiles). Hold the owning texture handle in a Python dict; LRU
eviction drops the handle and the framework frees the GPU
resource on the next frame. Cap the cache by entries × bytes-per
to bound GPU memory.

Synchronous GPU upload is sometimes mandatory: an async
"create-from-encoded-bytes" helper may return a zero texture_id
in the same frame, and the first draw call asserts. Use the
synchronous "create-from-decoded-rgba" path when you need a valid
texture in the same frame.

**Multi-context split (decode once, upload per context).** If
the UI runs more than one framework context (a separate login
modal with its own event loop, multiple windows with independent
GL contexts, an embedded preview that boots / tears down on
demand), GPU texture IDs minted in one context are **invalid**
in another. Split the asset lifecycle in two:

```python
# Module-scope cache of the DECODED PIXELS — CPU data, context-
# independent, shared across contexts.
_rgba: Optional[np.ndarray] = None

def _decode_once() -> np.ndarray:
    global _rgba
    if _rgba is None:
        _rgba = _decode_png(asset_path)
    return _rgba

# Each context mints its OWN GPU texture from the cached pixels,
# keeps the returned owning handle alive for that context's
# lifetime. When the context tears down, the handle drops and
# the GPU resource is freed.
def upload_for_this_context():
    return hello_imgui.create_texture_gpu_from_rgba_data(_decode_once())
```

A single module-level GPU handle would dangle the moment any one
context lifecycle ended; subsequent contexts would render
garbage or crash on a stale texture id.

<a id="input-arbitration"></a>
### 8.9 Input arbitration (safety footgun)

A driving-input listener (WASD for throttle/steering, gamepad
axes for control output) that consumes keys unconditionally is a
safety bug. A stray "w" typed into a filter box jams throttle.

Guard every input consumer with the framework's "is some widget
already capturing this input?" predicate:

```python
io = imgui.get_io()
if io.want_capture_keyboard:
    return                      # text widget owns the key
# else: consume for driving control
```

Apply this **before** the input-to-actuator path runs. This is a
safety rule, not a UX one — see [§3 Safety-critical changes](engineering-rules.md#safety-critical).

<a id="device-hot-swap"></a>
### 8.10 Target / device hot-swap

A UI that connects to remote targets must provide a single
"swap target" entry point. Build-new-first sequence:

1. Build the **new** transport. Don't tear down the old before
   the new is alive — avoids a service-less window where
   subscribers reconnect to nothing.
2. For each per-transport service: prefer `set_transport(new)` if
   the service supports hot-swap; otherwise `stop()` and rebuild.
3. Replace the shared transport handle.
4. Rebuild any supervisor whose tick thread is dead.

This runs on the UI thread (typically called from a button click
inside a frame). Teardown blocks briefly (<100 ms typical) — an
invisible blip, not a visible dropped frame.

<a id="orphan-sweep"></a>
### 8.11 Orphan sweep on startup

UIs that spawn child processes get reparented to PID 1 on
ungraceful exit (SIGKILL, terminal closed, crash). Children
linger, holding shared memory and (worst case) emitting network
traffic.

Run an orphan sweep at startup, **before** spawning new workers.
Conservative match: only kill a process whose cmdline contains
your exact project root, never the bare command name. Provide an
env-var override (e.g. `NO_ORPHAN_KILL=1`) for debugging.

<a id="ui-helpers"></a>
### 8.12 Cross-cutting UI primitive helpers

When you find 6+ views each rolling their own version of the same
primitive (first-frame focus capture, submit-on-Enter, an
async-action button that disables itself while in-flight, a
"click twice within 3 s to confirm" destructive button), extract
**one** helper. UI conventions change; you want one place to change
them, not 6 copies to chase.

<a id="fonts"></a>
### 8.14 Fonts and the symbol-glyph invariant

A UI that draws status dots, check / cross marks, arrows, degree
signs, units, and any non-ASCII text **must** load a font that
covers every glyph it draws. The default font many GUI frameworks
ship carries Basic Latin only; everything beyond renders as the
missing-glyph box ("tofu") in production.

Two rules keep this clean:

1. **Load one Unicode-wide font as the default.** Replace the
   framework's stock font loader with a callback that registers
   exactly one font covering Latin + CJK + box-drawing + geometric
   shapes + the symbols you use. Modern immediate-mode atlases
   rasterise on demand — no need to pre-declare glyph ranges.

2. **Centralise the symbols.** Every non-ASCII glyph the UI uses
   goes through a single `SYM` class (or module-level constants).
   Views reference `SYM.DOT`, not the literal `"●"`. The invariant
   is: every codepoint in `SYM` is verified present in the bundled
   font. A font swap is then a one-place vetting exercise, and a
   missing glyph fails review, not production.

```python
class SYM:
    DOT       = "●"   # filled status dot — live / OK
    DOT_OPEN  = "○"   # outlined status dot — stale / inactive
    CHECK     = "✓"   # success
    CROSS     = "✕"   # failure — NOT "✗" (some fonts only carry ✕)
    DEGREE    = "°"
    ARROW_UP  = "↑"
```

A glyph that isn't in the bundled font in *bold* / *italic* /
*large* variants is still a missed glyph — verify the actual
weight you'll render in.

**And in the actual theme.** A regular-weight font may have
acceptable contrast on a dark background and read as washed-out
gray on a light one. The "looks fine on my machine" pre-condition
is *dev-default theme*; production hits every theme the operator
can switch into. If your fix is to bump the font weight (Regular
→ Medium), bundle the heavier weight; the heavier file is small
relative to the readability bug.

**Variable fonts: instance to a fixed weight, don't ship the
variable file.** A variable font carries a `wght` axis that
defaults to its lowest value (often Thin / 100). A rasteriser
without a variable-axis engine (e.g. `stb_truetype`, as opposed
to FreeType) **can't apply the axis** — it renders the Thin
default, which on a wide-coverage CJK face looks gray and
washed-out at normal sizes. Symptom: "the text is too faint,"
chased as a theme/contrast bug when it's the font shipping at
weight 100. Fix: bundle a **static instance** at the weight you
want, not the variable file:

```sh
# instance a variable font to a fixed weight before bundling
fonttools varLib.instancer Source-VF.ttf wght=500 -o Source-Medium.ttf
```

Verify the bundled asset is the *instanced* file, not the VF — a
file-size/name check at build time catches a regression here.

<a id="i18n"></a>
### 8.15 Multilingual support (i18n)

Designing for more than one language is cheap if you do it from
the first string and expensive to retrofit. The whole discipline
is: **every user-facing string goes through one translation
helper, the source language is the key, and the layout + font are
built to absorb other languages.**

#### English is the msgid

Strings flow through one helper that returns the active-locale
translation of an **English source string**, falling back to the
source string itself when no translation exists:

```python
def tr(msgid: str) -> str:
    return _CATALOGS.get(_active_locale, {}).get(msgid, msgid)

# at call sites
imgui.text(tr("Connecting to robot…"))
imgui.button(tr("Cancel"))
```

Three properties make this the cheap design:

- Wrapping a string in `tr(...)` **never breaks rendering** —
  worst case you see the original English. A missing translation
  is degraded, not broken.
- English **is** the msgid; English is not a catalog. Only
  non-English locales carry dictionaries, so there's nothing to
  keep in sync for the source language.
- Adding a language is "append a catalog to the registry" — no
  code edits anywhere else. Catalogs are registry-extensible
  (`en`, `zh-CN`, … each a dict); a switcher in Settings selects
  the active one.

#### Wrap *every* user-facing string — at the moment you write it

`tr()` is worthless on the 80 % of strings someone remembered to
wrap. A single bare label in a dialog is an untranslatable hole.
Wrap as you write, not in a retrofit pass — retrofits always
miss some, and the misses are invisible until a non-English
operator hits that screen. (Same family as the uno-namespace
naming rule, [§1](engineering-rules.md#naming): user-visible strings are managed
centrally, not pasted at the call site.)

#### Don't build sentences by concatenation

Word order differs across languages; a sentence glued from
fragments is ungrammatical (or impossible) in some of them.
Translate the **whole** sentence with a placeholder, never
`tr("Connected to ") + name + tr(" robots")`:

```python
# WRONG — fragments can't reorder; some languages put the count last,
# the noun first, the verb at the end.
tr("Found ") + str(n) + tr(" robots")

# RIGHT — one translatable unit; the catalog controls word order.
tr("Found {n} robots").format(n=n)
```

If a count needs grammatical agreement (1 robot / 2 robots), that
plural rule belongs in the catalog layer, not in `if n == 1`
branches at the call site.

#### Layout must absorb text expansion

Translated text changes size: European languages run ~30 % longer
than English, CJK is shorter but **taller** and needs the wide
font. A label or button sized to fit the English string clips or
overflows in another language. Size to content, or reserve slack —
never hard-code a width that only the English string fits. This
is the multilingual case of the device-sizing rule
([§8.23](#device-sizing)): the string you developed against is not
the widest string the UI will render.

#### The font must cover every target script

A translation the bundled font can't render is tofu, not text.
The single Unicode-wide font from [§8.14](#fonts) must cover every
script you ship a catalog for (Latin + CJK + whatever's next).
Adding a language is therefore *two* steps: the catalog **and** a
glyph-coverage check against the bundled font.

#### One glyph vocabulary, even across processes

Non-ASCII glyphs go through the `SYM` constants ([§8.14](#fonts)),
never pasted literals. Components that **can't import** the shared
i18n module — a probe script running in a different interpreter /
venv, a remote-side tool — must emit the **same literal glyph the
shared font carries**, and any parser on the other end must match
that exact codepoint. A status dot rendered as `×` (U+00D7,
present in the font) in one process and `✗` (U+2717, absent) in
another is a tofu bug across the boundary.

#### No transliteration / ASCII-fallback layer

The fix for a missing glyph is **the font covers it**, not a shim
that swaps `●` → `*` or strips accents. An ASCII-fallback layer is
a second, lossy rendering path that drifts from the real one and
hides the actual gap (a missing glyph or a missing translation).
Delete it; make the font and the catalog complete instead.

#### Locale is read once, switched deliberately

The active locale persists to a per-user config file (or env var),
read at process start. A Settings switcher can change it live if
every visible string re-renders from `tr()` next frame (natural in
immediate mode); otherwise changing it requires a relaunch. Don't
hot-swap the locale mid-frame in a way that leaves half the UI in
the old language.

<a id="animation"></a>
### 8.16 Frame-rate-independent animation in immediate-mode

Immediate-mode UIs have no widget-owned timers or animation
clocks. Smooth transitions come from a per-call state object that
remembers the previously displayed value and eases toward the
target each frame, scaled by **the current frame's `delta_time`**:

```python
@dataclass
class Smoother:
    """First-order low-pass on a float. Frame-rate-independent."""
    value: float
    time_to_target: float = 0.25   # e-folding time constant (s)

    def step(self, target: float, dt: float) -> float:
        # alpha = 1 - exp(-dt/τ) is the dt-correct formulation
        alpha = 1.0 - math.exp(-dt / max(self.time_to_target, 1e-6))
        self.value += (target - self.value) * alpha
        return self.value
```

`time_to_target` is the e-folding time constant — after that many
seconds the smoother has covered ~63 % of the gap. The same
animation looks identical whether the loop runs at 60 Hz, 30 Hz,
or 120 Hz, because the formulation **doesn't depend on a fixed
per-frame step**.

Hold one smoother per widget instance — in a module-level
`get_smoother(key, time_to_target)` cache when the call site has
no natural place to stash state. Stale smoothers add constant
memory, not a leak.

`alpha = dt / time_to_target` is *not* correct — it's the linear
approximation and overshoots at low frame rates. Use the
exponential form above.

---

<a id="banner-pattern"></a>
### 8.18 The status banner pattern

A status banner (network health, version drift, registry
reachability, build pinning) follows a fixed shape. Memorise it
and re-use it; don't reinvent each one.

**Color triad + muted-init**:

| State | Colour | Meaning |
|---|---|---|
| Healthy | green | Source reports OK |
| Degraded but recovering | amber | Source reports a known-recoverable issue with a hint of what to do |
| Failure | red | Source reports an unrecoverable / actionable failure |
| Pre-probe / first frame | muted | No data yet — show **nothing** to avoid flashing on cold start |

```python
def draw_banner(snap):
    if snap is None:
        return                        # first frame, no data yet
    color, label, hint = _format(snap)
    imgui.text_colored(color, label)
    if hint:
        imgui.same_line(); imgui.text_colored(MUTED, hint)
    if imgui.is_item_hovered() and snap.detail:
        imgui.set_tooltip(snap.detail)
```

**Three more rules around the shape:**

1. **Hover = detail.** The tooltip carries the multi-line
   diagnostic the operator would copy into a bug report (egress
   interface, error code, raw exception text). Hovering is a
   diagnostic action, not part of the steady-state cost.
2. **Hidden when healthy** if "healthy" is the steady state and
   the banner exists to surface deviation. Don't burn a row of
   UI on "everything is fine."
3. **Rate-limit any background probe** the banner triggers — a
   per-frame imgui call at 60 fps must not fire an HTTP request
   at 60 Hz. Cache the result in a module-level scalar with a
   timestamp; refresh only when the timestamp ages out:

   ```python
   _LAST_CHECK_T = 0.0
   _LAST_RESULT: Optional[Snap] = None

   def draw_banner():
       global _LAST_CHECK_T, _LAST_RESULT
       if time.monotonic() - _LAST_CHECK_T > _CHECK_INTERVAL_S:
           _LAST_RESULT = _probe()      # background thread in practice
           _LAST_CHECK_T = time.monotonic()
       _draw(_LAST_RESULT)
   ```

<a id="wire-state-mirror"></a>
### 8.19 UI mirrors the value going on the wire, not the value being typed (safety)

This is the rule the entire control-input design hinges on. The
on-screen joystick / steering / throttle widget renders from the
**publisher's snapshot of what's actually being sent**, not from
its own local key/gamepad read:

```python
# WRONG — UI shows what the operator THINKS is happening
def draw_control(local_keys):
    crosshair.draw(local_keys.x, local_keys.y)

# RIGHT — UI shows what's actually on the wire
def draw_control(joystick_service):
    snap = joystick_service.snapshot()
    crosshair.draw(snap.axes[0], snap.axes[1])
    if snap.stale:
        imgui.text_colored(ERR, "wire silent — axes zeroed")
```

Three reasons:

1. **Source merging.** The publisher merges WASD + gamepad + web
   client into one stream. The UI must reflect the merge, not one
   input source.
2. **Watchdog state.** If the safety watchdog zeroed the axes (no
   input in N seconds), the operator must see zero on screen, not
   the last thing they typed.
3. **Local-vs-wire skew.** A bug where the local handler runs but
   the publish loop is dead produces a UI that looks fine and a
   vehicle that doesn't move. Reading from the publisher snapshot
   makes that visible immediately.

The corollary is an anti-pattern (#21) — any control widget that
reads its own input state for display is wrong by construction.

<a id="overlay-staleness"></a>
### 8.20 Overlays distinguish "alive but silent" from "broken"

A visualisation overlaid on live media (model-prediction polyline
on a video tile, pose marker on a map, GPS breadcrumb on a chart)
has three rendering states, not two:

1. **Fresh data** — draw normally.
2. **Stale data** (snapshot older than threshold) — render an
   explicit *amber-dashed "no data" indicator* over the same
   region. The viewer must be able to tell at a glance.
3. **No data ever received** — render the same "no data"
   indicator. Pre-first-frame is the same UX as gone-silent.

Without state (2), an operator watching an empty tile can't
distinguish "the overlay code is broken" from "the upstream
service went silent." Both are bugs; they need different fixes.

```python
def draw_overlay(snap, tile_rect):
    if snap is None or now() - snap.t > _OVERLAY_STALE_S:
        _draw_amber_dashed_no_data(tile_rect)
        return
    _draw_polylines(snap, tile_rect)
```

<a id="fault-localization"></a>
### 8.21 Live fault-localization diagnostics in the UI

When a control / status path crosses multiple components (input
device → publisher → transport → bridge → remote-side receiver),
operators can't tell from a single "not working" symptom *where*
in the chain it broke. Surface the per-hop state in the live UI:

```
Control panel:
  ┌─────────────────────────────────────────┐
  │ ⊙ Joystick:  ▶ axes [0.42, 0.00, 0.00]  │  ← local input
  │   sent:      ▶ tx 4231 frames           │  ← publisher
  │   rig:       ✓ receiving (last 0.08 s)  │  ← remote ack
  │   channel:   ✓ DC open · RTT 38 ms      │  ← transport
  └─────────────────────────────────────────┘
```

Each row collapses one hop's status. "Control not working" becomes
"control's leaving the GUI but rig says not receiving" in one
glance — pointing the operator (and the assistant debugging on
their behalf) at the bridge config, not the GUI parser.

This is the same idea as a "status banner" ([§8.18 status banner](#banner-pattern)) but applied to
the actual workload pane, not a header. Wherever a workload
crosses 3+ hops, the UI should let the operator localize the
fault in 5 seconds.

<a id="masking"></a>
### 8.22 Mask infrastructure detail and sensitive data on always-visible surfaces

A status row, banner, list, or panel that is **always on screen** —
and therefore always in screenshots, screen-shares, demos, and
bug-report photos — must not render raw **infrastructure detail**
(public IPs of edge nodes, internal LAN addresses, bearer tokens,
port numbers, SSH user@host) **or sensitive user data** (a bank
card / account number, an ID, a phone, an API key). The operator
shares their screen with a customer and leaks the fleet's network
map — or a person's bank details.

The rule:

- **Always-visible surface → masked / friendly form.** A hostname
  (`rig-07`), a role label (`LAN` / `TUNNEL`), a masked address
  (`113.x.x.11`), a masked secret (`•••• 4321`). Never the raw
  value.
- **Full detail lives in hover/tooltip or an expandable panel** —
  a deliberate action the operator takes when they actually need
  it, not something a passive screenshot captures.
- **Sensitive data is masked by default; revealing it is the
  explicit action, never the resting state.** A card/account/ID/key
  in a list shows `•••• 4321` until the user clicks "reveal" —
  least-exposure is the default. And never log the cleartext
  (that's the §16 / repo-secrets surface, one layer down).
- **Hiding it in the client is *display* hygiene, not access
  control — gate it at the API too.** A field a role isn't allowed
  to see must be withheld **server-side** (the endpoint doesn't
  return it / returns it masked for that role), not merely hidden
  in the UI — a hidden-but-served field is one DevTools tab or one
  `curl` away. UI masking and server-side authorization are two
  layers; ship both. (This is the read-side of
  [§3.1](engineering-rules.md#safety-critical)'s "control is dropped server-side, not
  by hiding the button.")

```python
# WRONG — raw IPs on a banner that's always visible
banner.text(f"{lan_ip} → {vps_public_ip}  {rtt}ms")

# RIGHT — shape on the banner, detail behind a hover
banner.text(f"{route_label}  {rtt}ms")        # "LAN 8ms"
if banner.hovered():
    tooltip(f"egress {lan_ip}\nvia {vps_public_ip}\n{rtt}ms")
```

This is distinct from secrets hygiene (§18 — don't *commit*
secrets). This is "don't *render* infra detail where a camera
will capture it." Both are leak-prevention; the surface differs.

When you add any always-on status element, ask: "if this is in a
customer screen-share, what did we just hand them?"

<a id="device-sizing"></a>
### 8.23 Size for the deployment context, not the dev desktop

A UI that runs on an **in-vehicle / kiosk / field screen** wants
large legible text and big touch targets. The dev laptop is not
the deployment context — what looks right at desktop density is
unreadable and un-tappable on the device. Pick the target's
scale and hold it; don't let a re-theme quietly shrink controls
back to desktop density.

Two coupled facts make this a rule, not a preference:

**1. Font size and control geometry are paired knobs — change
them together.** A bigger font with the old padding looks
cramped; bigger padding without the font looks empty. Bind them
in one place:

```python
FONT_SIZE = 20.0                 # device baseline; keep ≥ the floor

# geometry scaled to match the font — NOT independent magic numbers
frame_padding  = (14, 10)        # button / input hit area
item_spacing   = (14, 11)
window_padding = (16, 14)
grab_min_size  = 14              # draggable handles stay finger-sized
```

If you bump one, audit the other in the same change.

**2. When a size change invalidates a cached layout, bump the
cache key.** UI frameworks persist window/dock arrangements
keyed by a layout name (or version). If you change the font and
*don't* bump the key, existing users keep the old arrangement
computed for the old metrics — overlapping panels, clipped
status bars. Carry the metric in the key so it self-invalidates:

```python
layout_name = "default-20pt-alerts-overlay"   # bump the NNpt when FONT_SIZE changes
```

This generalises beyond fonts: **any persisted-and-cached layout
state must be keyed by the inputs that determine it**, so a
change to those inputs forces a recompute instead of restoring a
now-wrong cached arrangement.

**3. The code is the layout authority — the persisted file is a
cache, never the source.** This is the rule that stops a layout
overlap (status bar over docks, panels colliding) from coming
back **again and again** after you "fixed" it. The back-and-forth
cycle is always the same shape:

> You see overlap → you drag panels apart by hand → the framework
> writes your drag into the persisted layout file
> (`imgui.ini`/equivalent) → it looks fixed *on your machine* →
> you ship → a fresh machine (empty cache) computes the *code*
> layout, which still overlaps → overlap is "back."

You fixed the cache, not the layout. The code's programmatic
split usually only applies `first_use_ever`, so on every machine
that already has a cache file your code change is invisible, and
on every machine that doesn't, your hand-drag is invisible. The
two diverge forever.

The rule, ironclad:

- **Fix overlap in the programmatic layout (the dock-split code),
  never by dragging panels in a running build.** A hand-drag is a
  local cache mutation; it does not change what anyone else sees.
- **After changing the layout code, bump the layout key** (point
  2) so every existing cache is invalidated and the new code
  layout actually takes effect. A layout-code change without a
  key bump reaches *zero* existing users.
- **Verify the fix against a wiped cache, not your working one.**
  Delete the persisted layout file (`rm ~/.config/<app>/imgui.ini`
  or equivalent) and relaunch. If overlap is gone from that cold
  start, it's fixed for real; if it's only gone in your warm
  session, you fixed the cache.
- **Reserve fixed screen space for always-on chrome in the layout
  code.** A bottom status bar / top alert banner that overlaps
  the docks means the dock space wasn't told to leave room for
  it. Set the reserved height in the programmatic layout (e.g.
  hello_imgui's bottom-status / dockspace height), don't rely on
  the panels happening to not reach that far.

One-line test before you call a layout overlap fixed: **"would a
brand-new machine with no cache file see this fix?"** If the
answer isn't a confident yes, you changed the cache, not the
code.

<a id="framework-chrome"></a>
### 8.24 Disable framework default chrome when you own that surface

UI frameworks ship default chrome — an app menu, a view menu, a
status bar, a window title — often enabled by default. If you
render your **own** version of that surface, the framework's
default renders *alongside* yours: a duplicate "View" menu, a
stray app menu, two title rows.

When you take ownership of a surface, explicitly disable the
framework's built-in:

```python
# we own the whole menu bar via our own draw_menu_bar()
params.show_menu_app  = False     # else a duplicate app menu appears
params.show_menu_view = False     # else a duplicate "View" menu appears
```

The rule: **owning a surface means disabling the framework's
default for it, not just drawing over it.** Drawing over it
leaves both. And when you draw custom content inside a
framework-managed bar (a logo image in the menu bar), snapshot
and restore any cursor/layout state you perturb (`cursor_pos`
around an `image()` + `same_line()`), or the framework's own
next item renders misaligned.

---

<a id="theme-colors"></a>
### 8.25 Read colours from the active theme, never hardcode them

A UI that supports more than one theme (dark + light, high-contrast,
day/night) must take every colour from the **active style**, not a
literal. A hardcoded colour that looks right on the theme you
developed against is illegible on the other — green text on a dark
panel becomes invisible on a light one; a status line painted a
fixed colour reads as "ugly and wrong" the moment the theme flips.

```python
# WRONG — fixed colour; correct on one theme, broken on the other
draw_chip(text, color=(0.2, 0.9, 0.2, 1.0))          # green — unreadable on light

# RIGHT — pull from the active style so it tracks the theme
style = imgui.get_style()
draw_chip(text, color=style.color_(imgui.Col_.text),
                border=style.color_(imgui.Col_.border))
```

Three rules:

- **Semantic colours come from one named palette, switched with
  the theme.** `OK / WARN / ERR / MUTED` are defined per theme
  (a registry of `ImVec4`s mutated in place on switch), not
  written as literals at each call site. A theme change updates
  the palette in one place; every consumer follows.
- **A status indicator's *off / false / absent* state is per-site
  semantic — declare what it means.** The same `False` is *normal*
  at one indicator (an optional feature that's off, telemetry that's
  quiet → `MUTED` gray) and a *fault* at another (a link or control
  authority that should be live → `ERR` red), or merely *noteworthy*
  at a third (not-yet-engageable → `WARN` amber). A shared
  status→colour helper takes the off-meaning **per call site**
  (`off=IDLE|WARN|BAD`); a single fixed off-colour mislabels half of
  them — a down link painted neutral gray reads as "fine" and the
  operator **misses the fault**, while a normally-absent signal
  painted red cries wolf. This is operator-truth
  ([§8.19](#wire-state-mirror)): the indicator means what the operator
  needs *at that spot*, not a uniform default. (The colours still come
  from the one palette above; only the off-*semantic* is per-site.)
- **Don't colour a whole region to signal one state.** Painting
  an entire line/banner green for "healthy" is the pattern that
  breaks across themes and overwhelms the eye. Carry state in a
  small dot / chip / icon against the theme's own background, so
  the surface stays legible on every theme.

This is the colour analogue of the font/glyph invariant
([§8.14](#fonts)): a UI element must render correctly under every
theme the build ships, verified in each — not just the one you
had open.

<a id="qt-legacy"></a>
### 8.26 Retained-mode (Qt) — addendum for legacy maintenance

The snapshot pattern above is the **default house style for new
UIs**. The retained-mode rules below apply when you're maintaining
or porting a Qt UI. They're not deprecated, just legacy.

#### Never close from a worker thread

`widget.close()` from a worker thread can deadlock on macOS
because the main loop owns the widget tree. Bounce back to the
GUI thread:

```python
QMetaObject.invokeMethod(
    dialog, "done", Qt.QueuedConnection,
    Q_ARG(int, 0 if ok else 1),
)
```

#### Don't `processEvents()` spin-wait

The canonical async progress-dialog pattern:

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

#### Don't pin the asyncio loop with CPU work

Heavy work like image decoding inside an asyncio coroutine must
run in an executor; otherwise it pins the loop for hundreds of ms
per second and every other subscriber on that loop stalls:

```python
img = await loop.run_in_executor(
    None, lambda: frame.to_ndarray(format="rgb24")
)
```

#### One cancellation primitive per project

Don't let workers each invent `_stop` / `_cancel` / `_abort` —
naming drift across cancellation protocols is the smell that
forces dialogs to introspect signal names by string in
`closeEvent`. Pick one (a `threading.Event`, a single `stop()`
method) and impose it on every worker via a base class or
registration step.

<a id="verify-after-write"></a>
### 8.27 A control action confirms by reading back actual state, never by "the write returned OK" (safety)

[§8.19](#wire-state-mirror) is the steady-state rule — a *continuous*
widget renders the wire snapshot, not local input. Its discrete
cousin: a **control action** (a button that engages, arms, sets a
mode, pushes a setting) must show the **state it reads back from the
authority**, not an optimistic latch on the write's return code. *The
write succeeding is not the target accepting it* — and a status
indicator that says success the instant the call returns will **lie**
whenever the target rejects the value.

We shipped this: an engage pill latched `self._engaged = True` on a
zero return from the engage write, and reconciled only every 2 s. But
the target could reject the engage **same-tick** — a blocking safety
event was active, or a manual-lock re-latched — so the value never
took. The pill showed **"ENGAGED" while the vehicle sat in STANDBY**
and never actuated: a safety-state indicator asserting a state the
system was not in.

- **Verify-after-write.** Write, then **read the value back** (ideally
  in the same round-trip) and reflect the **read-back**, not the
  request. A same-tick reject then shows as a reject, immediately —
  log a warning when the write didn't take.
- **Reconcile-poll for *later* clears.** Verify-after-write catches a
  same-tick reject; a value the target drops *later* (a new blocking
  event, an external re-latch) is caught only by a periodic
  reconcile read ([§25.1](delivery-rules.md#reconcile-poll)) — keep its period short
  for a safety-relevant state.
- **This is [§3.5](engineering-rules.md#validate-safety-input)'s "verify it took effect on
  the target" at the UI:** "issued" is not "applied," for a control
  write exactly as for a deployed calibration.

The generalisable rule: **a control action's confirmation reflects a
verified read-back of the authority's actual state, never the fact
that the write returned success — an optimistic latch lies the moment
the target rejects the value (a same-tick block, a re-latch), and a
safety-state indicator must never assert a state the system is not
in.** See anti-pattern #75.

<a id="house-design-language"></a>
### 8.28 Commit to one named visual design language — the house style is Song-minimalist (Suzhou-garden)

A UI's *look* is a contract too. **Pick one named visual language and
define its tokens once** — palette, type scale, spacing, line weight,
motion — in the single theme module every surface reads from
([§8.25](#theme-colors)); hold every screen to it. A screenshot should
be recognisable as *ours* without a logo. Without a named language and
one token source, each view drifts into ad-hoc colours, weights, and
spacing — the product looks assembled by strangers, and "make it look
right" decays into per-widget whack-a-mole.

**The Unomove house language is Song-minimalist** — the restraint of
Song-dynasty aesthetics and a Suzhou garden: **ink on rice-paper,
hairline structure, one quiet accent, and generous 留白 (negative
space).** It's a positive org convention, like the
[uno-namespace](engineering-rules.md#naming) and
[`unolib`](engineering-rules.md#reuse-critical-path): a new GUI
**adopts** it, it doesn't invent its own.

The principles *are* the rule (the hexes below are reference, not law):

- **留白 — negative space is structure, not emptiness.** Lead with
  whitespace and asymmetric balance; separate with **hairline (1px)
  rules**, not boxes or drop-shadows. Fewer elements, more room. If a
  screen feels full, *remove* — don't shrink.
- **Ink on paper.** A warm rice-paper surface, ink-grey text, muted
  natural mid-tones (celadon, tile-grey, bamboo, water). Low
  saturation throughout — nothing shouts.
- **One accent, used like a seal.** A single cinnabar/seal-red (朱砂)
  marks the *one* most important action or live state on a screen. If
  everything is accented, nothing is.
- **Quiet typography.** A Song-style serif (Songti / Noto Serif CJK)
  for headings, a clean sans for body; generous line-height; hierarchy
  carried by weight and size, not colour.
- **Calm motion.** Slow, eased, sparing — a brushstroke, not a spinner.

Reference tokens (authoritative values live in each GUI's theme module,
e.g. a `theme.py`; a dark **ink-night** variant keeps the same
restraint for glare / outdoor use):

| token | rice-paper (light) | role |
|---|---|---|
| `paper` | `#F4EFE4` | surface / background |
| `ink` | `#2A2A28` | primary text |
| `ink_muted` | `#6B6B63` | secondary text |
| `line` | `#CFC8B8` | hairline dividers / borders |
| `tile` | `#5A6066` | structural grey (黛瓦) — headers, rails |
| `celadon` | `#8AA79B` | calm / healthy state (青瓷) |
| `bamboo` | `#6E8B74` | secondary natural accent |
| `water` | `#9FB1B5` | info / inactive |
| `cinnabar` | `#A8423A` | **the** accent (朱砂) — one per screen |

The generalisable rule: **adopt one named visual design language and
source its tokens from a single theme module; restraint and negative
space are deliberate features, not absences. The Unomove house
language is Song-minimalist / Suzhou-garden — ink on rice-paper,
hairline structure, one cinnabar accent, generous 留白 — and a new GUI
inherits it instead of improvising.**

<a id="rerunnable-action"></a>
### 8.29 An operator action that drives a flow is re-runnable — clicking it again is the recovery path

The operator's universal recovery move is **to do it again** — re-press
Onboard, Link, Enroll, Deploy, Retry. So a button that kicks off a
multi-step flow (a bootstrap, a link/enroll, a deploy, a bump) must be
**safe to re-run**: a second press **converges** — it skips what's
already done and retries only what failed — rather than erroring,
double-applying, or wedging. A one-shot action that leaves a half-done
state on the first failure (a partial onboard, a link that committed
locally but failed to publish) and then **refuses or double-applies on
the second press** forces the operator to a terminal and someone with
CLI access — exactly what a field operator doesn't have.

This is the GUI face of [§18.7](delivery-rules.md#finish-the-operation)
(finish the operation; idempotent skip-check on the published state —
anti-patterns #87, #88). At the operator surface that discipline *is* the
lifeline:

- **Idempotent + converging on re-press.** Re-running checks the
  **end-state each step targets** (the live value, the pushed ref, the
  registered entry — via [§8.27](#verify-after-write) read-back), does the
  steps not yet true, and leaves the done ones alone. "Already linked" is
  a clean no-op, not an error and not a second link.
- **Re-run *is* the recovery path — one click, always available.** Don't
  hide the action after the first attempt or gate re-link behind "unlink
  first." A failed/partial run leaves the button live and labelled for
  what it will do ("Re-link", "Retry onboard").
- **Show the verified end-state, not "submitted."** Reflect the state the
  flow actually reached ([§8.27](#verify-after-write)) and name *which*
  step is still outstanding, so the re-press is informed, not blind.
- **Auto-attempt on open where it's safe.** A flow whose natural state is
  "should already be done" (link-on-startup, reconnect) re-runs itself
  when the surface opens — idempotency is exactly what makes that safe
  ([§6.3 clients reconnect across restarts](engineering-rules.md#always-on-contract)).

The generalisable rule: **an operator action that drives a multi-step
flow is idempotent and re-runnable — a second press converges (skips
done, retries failed), is the one-click first-line recovery (always
available, labelled for what it does), and reflects the verified
end-state — never a one-shot that strands a half-done state the operator
can fix only from a CLI.** See anti-pattern #89.

<a id="rerun-visualization"></a>
### 8.30 For multimodal / time-series telemetry & debug visualization, log to Rerun — don't hand-roll a viewer

When you need to *see* what a robotics / perception / sensor pipeline is
doing over time — camera frames, point clouds, detections, transforms,
scalar plots, tensors, joint states — the reflex is to hand-build a
one-off viewer (a bespoke imgui/Qt panel, a matplotlib dump, a custom 3D
widget). Don't. The house tool for **introspection / telemetry / debug
visualization is [Rerun](https://github.com/rerun-io/rerun)**
(`rerun-io/rerun`) — the time-aware, multimodal log-and-view layer for
physical-AI data. It's a positive org convention like
[Rust](engineering-rules.md#rust-vs-python) (§9),
[`unolib`](engineering-rules.md#reuse-critical-path) (§28), and the
[Song-minimalist look](#house-design-language) (§8.28): a new debug
surface **adopts** it instead of reinventing a viewer.

- **Log, don't render.** Instrument the pipeline with
  `rr.log(entity_path, …)` — images, point clouds, boxes, transforms,
  scalars, tensors — and let the Rerun viewer draw, scrub time, and
  correlate streams. *Logging* (in your code) is separated from *viewing*
  (the viewer): the same producer/consumer split as the house snapshot
  pattern ([§8.2](#snapshot-pattern)) — your code emits data down an
  entity-path tree, the viewer is downstream.
- **One viewer, every modality, on a shared timeline.** 3D scene +
  images + plots + tensors + text logs, all indexed on the **one shared
  clock** ([§2.1](engineering-rules.md#timestamp-contract)) so they
  scrub and replay *together* — not N disconnected tools you eyeball
  side-by-side.
- **Live or replayed — but Rerun never *persists*.** `connect_grpc()`
  to a running viewer for a live session, or replay the **canonical
  record** — the unomsg `.log` ([§16.9](delivery-rules.md#one-record-format))
  — into a viewer with a `ulog-rerun`-style decoder. **Never
  `rr.save("…​.rrd")`:** a `.rrd` on disk is a *second* log format
  beside the one of record, which drifts (the §28 two-copies trap). The
  diagnostics artifact you capture on the rig and view later
  ([§15](delivery-rules.md#diagnostics), instead of a reflexive reboot
  [§15.7](delivery-rules.md#no-reflexive-reboot)) is the **`.log`**, not
  an `.rrd`; Rerun is the lens on it, live or replayed.
- **Fits the stack.** Rust core, SDKs in Rust / Python / C++
  (Rust-first, §9), Apache-2.0 / MIT. Reused, not reinvented (§28).

**Scope — Rerun is the introspection layer, not the control layer.** It
does **not** replace the operator/control GUI: engage pills, safety
state, anything on the wire-state-mirror / verify-after-write path
([§8.19](#wire-state-mirror), [§8.27](#verify-after-write)) stays in the
house control surface ([§8.2](#snapshot-pattern),
[§8.28](#house-design-language)). Reach for Rerun to *see and debug* the
data; reach for the control GUI to *operate*.

The generalisable rule: **for multimodal, time-indexed telemetry and
debug visualization (sensors, perception, 3D, plots), log to Rerun
(`rerun-io/rerun`) and let its viewer render / scrub / replay — a
positive house convention like Rust, `unolib`, and the house look —
instead of hand-rolling a one-off viewer; keep operator/control surfaces
in the house GUI, Rerun is the introspection layer, not the control
layer.** See anti-pattern #90.

---

## UI anti-patterns — the shapes that bite UI work

The full catalogue is in [`anti-patterns.md`](anti-patterns.md) (one
flat, stably-numbered list shared with the code rules). These are the
entries that recur in UI work — when your change *looks like* one,
open it there by number:

- **#1** — UI panel dead / stuck: the **bridge isn't forwarding its
  topic**. The panel is fine; check the publisher / forward set first.
- **#3** — `widget.close()` from a worker thread → macOS deadlock.
  Queue the close to the UI thread.
- **#13** — Unbalanced framework stack on exception: error recovery
  unwinds to the view's start depth (a `@safe_view` snapshot or
  try/finally), it doesn't assert.
- **#14** — Input listener consumes keys while a text field has focus
  — a safety footgun. Yield global handling when an editable has focus.
- **#21** — UI reflects local input, not the value on the wire: render
  the publisher snapshot ([§8.19](#wire-state-mirror), safety).
- **#24** — Raw infra detail / sensitive data on an always-visible
  surface: mask by default ([§8.22](#masking)).
- **#26** — Layout overlap that keeps coming back: you fixed the
  cache, not the code — bump the key, verify **cold**
  ([§8.23](#device-sizing)).
- **#35** — Translated / non-Latin text renders as boxes, or all text
  looks thin & gray: font coverage + static-instance weight
  ([§8.14](#fonts), [§8.15](#i18n)).
- **#75** — Control widget showed "engaged" on the write returning,
  not on a read-back ([§8.27](#verify-after-write), safety).
- **#89** — A one-shot operator action strands a half-done state;
  re-clicking errors ("already exists") or double-applies, so recovery
  needs a CLI. Make the flow idempotent and re-run the one-click recovery
  path ([§8.29](#rerunnable-action)).
- **#90** — Hand-rolled a one-off viewer (matplotlib dump, bespoke 3D
  widget) to debug a sensor/perception pipeline — N disconnected tools,
  no time-correlation, no replay. Log to **Rerun** instead
  ([§8.30](#rerun-visualization)).

## UI review checklist

The UI checklist items live in
[`review-checklist.md`](review-checklist.md) under **§ F. UI** (with
masking in § D and fonts / i18n alongside). Run that section over any
UI diff. In one breath: control widgets render from the **wire
snapshot** (not local input); a control **action confirms by
read-back**, not write-OK; status banners follow the muted →
green/amber/red **triad**; overlays distinguish **"alive but silent"
from "broken"**; sensitive data is **masked**; layout is verified
against a **wiped cache**, not a warm session; translated text has
**font coverage**; and global input handling yields when an editable
has focus.

---

