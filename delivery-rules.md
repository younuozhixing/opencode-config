# Delivery & operations rules

Companion to [`engineering-rules.md`](engineering-rules.md) — the **second half**
of the rule book, split out for size (the book crossed its own ~2500-line
threshold). These are **§11–§27**: parts **E build / deploy / fleet**, **F
diagnose & verify**, **G working safely**, **H integration & external
surfaces**. **Nothing was renumbered** — the sections keep their §11…§27
numbers and every slug anchor, so existing cross-references keep resolving;
`engineering-rules.md` carries a one-line stub for each. References to
*other* sections point at [`engineering-rules.md`](engineering-rules.md)`#…`.

Same conventions as the main file: stable numbers, slug anchors, a
`> When this fires … / Do …` header per section. Jump by task via the
trigger index in [`CLAUDE.md`](CLAUDE.md); scan every rule in one line via
[`quickref.md`](quickref.md).

---

<a id="cross-compile"></a>
## 11. Cross-compile

> **When this fires** — building or freezing for a different architecture / target.
>
> **Do** — cross-build with `zigbuild`; install every native dep in the build env; verify the linked arch and by importing entry points — not the exit code.

Cross-build for embedded targets via `cargo-zigbuild` (or the
equivalent zig-backed toolchain for the language at hand), not
Docker. The host runs the build; the target gets a binary that
matches its libc/sysroot without a container image to maintain.

Artifacts land in `dist/` (or the project's release directory). If
you reintroduce a Dockerfile cross-build, you owe the user a
specific reason.

<a id="platform-libs"></a>
### 11.1 Link the shared library built for the target platform — never mix

A compiled shared library is **platform-and-arch specific.** A
build for one platform must link the libs built for *that*
platform — macOS links the macOS `.dylib`, Linux links the
Linux `.so`, Windows links the `.dll` / `.pyd`; and within one
OS, arm64 links the arm64 build, x86_64 the x86_64 build. They
are not interchangeable and must never be mixed into one bundle.

The failure modes are unforgiving and often *silent until runtime*:

- Wrong OS format → the loader rejects it (`wrong ELF class`,
  `mach-o … but wrong architecture`, `%1 is not a valid Win32
  application`) — at least this one is loud.
- Right OS, wrong arch → `image not found` / `incompatible
  architecture`, or it loads and crashes deep in the first FFI
  call.
- A lib built against a different libc / framework version →
  links fine, then aborts on a symbol that isn't there at the
  deployed version.

This is the same root cause as the rsync clobber (§12,
anti-pattern #4) — a wrong-platform native artifact reaching a
target — but stated as the **positive rule that prevents it**:
the right artifact must be *selected by platform*, not whichever
one happened to be in the tree.

How to keep it correct:

- **Select the lib by platform + arch at build/bundle time.** A
  per-platform layout the packager picks from — never a single
  `lib/` that the last build populated:

  ```
  wheels/macos-arm64/…   wheels/macos-x86_64/…
  wheels/linux-x86_64/…  wheels/linux-aarch64/…
  ```

  The bundler reads `(os, arch)` and pulls the matching set; it
  does not glob a shared directory.

- **Don't commit a built `.so`/`.dylib`/`.dll` into the source
  tree as "the" lib.** It's correct for exactly one platform and
  wrong for every other (the §13 source-vs-deployed boundary:
  generated, gitignored, selected at package time — not vendored
  into source where a cross-platform build will grab it).

- **Build the lib *on* (or *for*) the platform that will run it.**
  Native build on the target, or a cross-build whose toolchain
  targets the right `(os, arch, libc)` triple (§11). A lib built
  on the dev laptop is the dev laptop's lib, not the target's.

- **Verify what got linked, don't assume.** `file lib*.dylib`,
  `lipo -archs` (macOS), `file lib*.so` / `readelf -h` (Linux),
  `otool -L` / `ldd` for the dependency chain. A one-line check
  in the build/selftest that the bundled lib's arch matches the
  target catches a mismatch before it ships, not at the
  operator's first crash. (The frozen-bundle selftest tier, §18,
  is the right home for it.)

The generalisable rule: **a binary artifact is valid only for the
exact `(os, arch, libc/runtime)` it was built for; a
cross-platform product carries one set per platform and selects
by platform at package time.** Mixing is not "usually fine" — it
fails, often silently, always on someone else's machine. See
anti-pattern #36.

<a id="freeze-native-deps"></a>
### 11.2 A freezer bundles only what it can import — a missing native dep silently drops the whole module

A freezer / bundler (PyInstaller, py2exe, and friends) discovers
what to ship by **importing and analyzing** your code. When a
module raises at *analysis* time — because a compiled extension it
pulls in `dlopen`s a **native shared library that isn't installed
in the build environment** — the bundler doesn't fail; it
**silently drops that module** and everything that needed it. The
build "succeeds," and the frozen app dies at first run on a clean
target with an `ImportError` for something you obviously depend on.

We shipped this: a messaging extension `dlopen`s `libzmq.so.5`; the
build image didn't have it, so `import <msg-lib>` raised during
analysis, the bundler dropped the compiled extension, and the
frozen app failed `import <transport>` — the *entire transport
layer* gone, from one absent `.so`.

Keep it correct:

- **The build environment carries every native runtime dependency**
  the bundled extensions `dlopen` — not just the Python packages.
  The `.so` must be present both at *collect* time (so it's analyzed
  and bundled) and at *run* time. List them explicitly in the
  image/recipe with a note on **why each is required** (which
  extension needs it), so the next person doesn't prune a
  "spurious" dep and silently delete a subsystem.
- **Trust import, not the exit code.** A clean freeze is not a
  working one. Verify by **importing the critical entry points in
  the frozen artifact** on a clean machine (`<frozen-app> -c "import
  <transport>, <core modules>"`, or a smoke selftest, §18.1) — the
  freezer's "could not import / module not found" warnings are easy
  to miss in build noise, so assert on the import, not the log.
- **Install the *minimal* form that satisfies the link, not a full
  driver.** When the extension links a vendor-pluggable runtime (an
  OpenCL / Vulkan ICD, a GPU stack) but your code path doesn't use a
  real device, the **loader/stub** package alone — the one that
  ships the `.so.1` soname (e.g. an ICD *loader*) — satisfies the
  import and bundles fine; it just reports zero devices, which is
  correct when you never open one. You don't need the full vendor
  driver in the build (or on the target) to clear the analysis-time
  link.

The generalisable rule: **a freezer ships only what it can import at
analysis time, so a missing *native* dependency silently removes a
whole module from the artifact — install every `dlopen`ed `.so` in
the build environment and verify the frozen app by importing its
entry points, never by "the build passed."** See anti-pattern #50.

<a id="cross-build-matrix"></a>
### 11.3 Drive cross-platform builds from a declared target matrix — and build for the real target early, it's a verification surface

Cross-platform building is a **process**, not a per-machine
improvisation. Two disciplines keep it systematic:

**1. A declared target matrix, one toolchain per target.** Name the
targets your product ships to — `host`/dev (e.g. macOS-arm64), `rig`
(aarch64-linux), `server` (x86_64-linux) — and select each with **one
command** that activates the matching toolchain + sysroot (`build
--target rig`, an `activate <target>` step), never an ad-hoc pile of
`--target` / `CC` / `SYSROOT` flags assembled differently on each
machine. That selection is a [pinned build input (§27)](#build-fix): the
same `(triple, sysroot, linker, feature set)` every time, on any host.
And build (and **publish**) the **whole matrix**, not just the host
you're on — a single-arch artifact strands every other target
([§28.1](engineering-rules.md#roll-out-shared-fix): every publish ships x86_64 *and*
aarch64).

**2. Building *for the target* is a verification surface — run it early
and often.** The host build is **not representative**: target-specific
issues compile clean on the host and surface only when you build for the
real `(arch, libc, OS)` — a `#[cfg]` / conditional-compilation branch
that's only active on the target, a struct or syscall layout that differs
by arch (pointer width, endianness, padding, FD-vs-classic frame
structs), a libc / kernel-header feature the host's headers expose but
the target's don't. We caught a CAN FD-vs-classic split exactly this way
— it built fine on the dev host and was **found only by cross-compiling
to the rig target**. So cross-build to every target **continuously** (in
CI / pre-merge), not once at release; the target build catches what the
host hides, and the [on-target *run* (§13.1)](#dev-vs-deployed-layout)
catches what even the target *build* doesn't.

The generalisable rule: **drive cross-platform builds from a declared
target matrix selected by one reproducible toolchain-activation command
(pinned inputs, the whole matrix every time), and treat building for the
real target as a verification surface run early and often — the host
build compiles target-specific code (cfg branches, ABI / struct layout,
libc features) clean, and it breaks only on the target's triple.** See
anti-pattern #97.

<a id="relocatable-rpath"></a>
### 11.4 Bake a relocatable rpath so the binary finds its libs — not an `LD_LIBRARY_PATH` crutch

A binary that links a shared library (`.so` / `.dylib`) has to **find
that library at runtime**, not just at link time. The brittle way is to
make every launch set a lib-path env var — `LD_LIBRARY_PATH`,
`DYLD_FALLBACK_LIBRARY_PATH` — so the loader stumbles onto the lib. That
is an [env-var crutch (§7.5)](engineering-rules.md#env-var-config): it lives *outside* the
artifact, differs per machine, and the binary dies with `image not
found` / `cannot open shared object file` the moment it runs without that
magic environment — a systemd unit, a cron job, another operator, the
deployed rig.

Bake the search path **into the binary** so the artifact is
**relocatable** — it runs wherever it lands:

- **Link with an rpath relative to the binary** — `$ORIGIN/../lib`
  (Linux) / `@loader_path/../lib` (macOS) — so a binary and its libs move
  together and resolve with **no** environment. Set it at link time
  (`-Wl,-rpath,…`, the toolchain's rpath flag, or a consume-time
  `activate` step that injects it), not after the fact.
- **`patchelf` / `install_name_tool` after the build is a fallback, not
  the plan.** Rewriting an already-built binary's rpath works in a pinch,
  but the durable fix is to *build* it relocatable; a post-hoc patch is
  one more step to forget on the next build ([§13.4](#regenerated-location)).
- **Link the published prefix's soname.** The prefix is the validated,
  rpath-wired artifact ([§13.2](#depend-on-prefix)) — resolve the
  published lib, not a copy in your source tree.
- **Verify from a clean shell.** Run the artifact with **no**
  `LD_LIBRARY_PATH` / `DYLD_*` set, on the deployed layout
  ([§13.1](#dev-vs-deployed-layout)); `ldd` / `otool -L` must resolve
  every lib via the rpath, not report "not found."

The generalisable rule: **a deployed binary finds its shared libraries
through a baked, relocatable rpath (`$ORIGIN` / `@loader_path`, or an
activation step that sets it) — never a runtime `LD_LIBRARY_PATH` /
`DYLD_*` env var (an env-var crutch, §7.5) or a forgotten post-hoc
`patchelf`; verify by running from a clean environment on the deployed
layout.** See anti-pattern #99.

---

<a id="embedded-sync"></a>
## 12. Syncing source to an embedded target

> **When this fires** — rsyncing source to an embedded target, or deploying across many rigs.
>
> **Do** — exclude native-arch binaries from the sync; stamp `git_sha`+dirty and diff stamps before debugging; gate compatibility on feature-set ⊆ advertised, not a version string.

When the host architecture differs from the target's, **exclude
native build artifacts from the rsync**. The host-built binary
silently clobbers the target-built binary; the daemon then runs
without errors and produces nothing useful (zero frames, empty
appsinks, dead RPC handlers).

Standard exclusion shape:

```sh
rsync -avz \
    --exclude='*.o' --exclude='*.os' \
    --exclude='__pycache__' --exclude='.venv' \
    --exclude='<path/to/each/native-arch binary>' \
    ./ <user>@<target>:<remote-path>/
```

List the binaries explicitly. Don't trust `*.bin` to catch
them — most native binaries have no extension.

<a id="build-version-stamp"></a>
### 12.1 Stamp the build; make every target report its version

Across a *fleet* of targets — many rigs, each rsync'd / flashed /
deployed to independently — the question that wastes the most time
is **"is this rig actually running the code I think it is?"** You
fix a bug, push, deploy to some rigs, and then chase a "still
broken" report that's really "this rig never got the new build."
Without a version you can read off the running system, every
cross-rig bug starts with un-answerable ambiguity.

The fix: **a build stamp the running system reports.** Wire it
once; it pays for itself on the first "works on rig A, not rig B."

**1. Stamp at build/deploy time.** Capture an unforgeable-ish
identity of exactly what was deployed — not a hand-bumped version
string that lies. Git is the cheapest source of truth:

```sh
# at build or deploy time, write a stamp into the artifact
cat > build_version.json <<EOF
{ "git_sha": "$(git rev-parse HEAD)",
  "git_dirty": $([ -z "$(git status --porcelain)" ] && echo false || echo true),
  "built_at_unix": $(now),     # pass in; don't bake a clock call
  "built_by": "$(whoami)@$(hostname)" }
EOF
```

`git_dirty: true` is the single most useful field — it catches
"deployed from an uncommitted working tree," which is how a rig
ends up running code that exists on no commit anywhere.
**Compute dirty with `git status --porcelain`, not `git diff`.**
`git diff --quiet` only sees **unstaged tracked** changes — it
reports *clean* for a **staged** edit or a **new untracked** file,
so a stamp built that way collides with the last commit's stamp
even though the tree differs. `git status --porcelain` catches
staged + unstaged + untracked in one check; that completeness is
what makes the stamp safe to **gate** a deploy on ([§12.3](#deploy-idempotent)).

**A clean / release build gates on the dirty flag — don't ship
`-dirty`.** The dirty flag isn't only for after-the-fact diffing;
it's a **release gate**. A `--clean` / release build path should
**refuse** (or require an explicit, loud `--allow-dirty` override)
when the source stamp is `-dirty`, because a `-dirty` artifact is
**unreproducible** — it embeds local edits that exist on no commit,
so "rebuild this exact binary from git" is already impossible and
"works on A, not B" can't be resolved by a sha diff. A development
build may be dirty; a thing you *deploy to a rig or publish* must
trace to a commit.

**2. The running system reports the stamp.** A `--version` flag, a
startup log line, a status topic field, a `GET /version` endpoint
— at least one. The operator (and the assistant debugging for
them) reads the *running* version, not the version they assume:

```sh
ssh <rig> '<app> --version'        # → git_sha, dirty, built_at
# or: grep the stamp in the startup journal
```

**3. Compare across rigs before debugging a discrepancy.** "Works
on A, not B" → first command is *diff the two stamps*, not read
code. Same shape as the [diagnostic-first rule (§15.1)](#diagnostic-first): the stamp
diff decides "same code, real bug" vs "different code, deploy
gap" in one step.

**4. Make staleness visible, don't rely on memory.** A deploy
script that records "rig X ← sha Y at time Z" (a fleet ledger),
or a startup check that warns when the running sha differs from
the latest released sha, turns "I think they're all updated" into
a fact you can query. Silent version drift across a fleet is the
same family as the layout-cache and dependency-lock traps: the
authoritative state and the deployed state diverge unless
something makes the gap loud.

The generalisable rule: **anything deployed to more than one place
carries a build stamp it reports at runtime, and "is it the right
version" is a command, not a belief.** See anti-pattern #29.

<a id="version-contract"></a>
### 12.2 Negotiate compatibility by capability, not by version string

The build stamp (§12.1) answers *"which build is this?"*. A separate
question — *"can this consumer talk to that producer?"* — must **not**
be answered by comparing version strings across repos. Two
independently-released components (a GUI and the service it calls; a
rig stack and the daemon it drives) each carry their own version, and
the moment one bumps without the other, a hand-mirrored
`EXPECTED_PEER_VERSION` constant in the consumer drifts and lies.

The failure is always the same shape: the consumer warns
`peer reports 3.2.0, expected 3.1.1` against a **newer, fully working**
peer. A newer peer is forward-compatible, so a version compare reports
a non-problem as a fault — and forces a cross-repo constant to be
hand-synced every release.

**Negotiate on capabilities instead:**

**1. The producer advertises a contract.** Its `/version` (or
`--capabilities`, or a status field) carries
`{service, version, features:[…]}` — `features` being the named
capabilities other components depend on.

```jsonc
GET /version → { "service": "ts-enroll", "version": "3.2.0",
                 "features": ["registry", "commands", "upload_relay"] }
```

**2. The consumer requires a feature SET, never a version.** Compatible
iff `required ⊆ advertised`. Extra features on the peer are fine
(forward-compat). When you start depending on a new endpoint, add its
*name* to the required set — that set **is** the contract.

```python
REQUIRED = {"registry", "commands", "upload_relay"}
ok = REQUIRED <= set(info.features)        # NOT info.version >= "3.2.0"
```

**3. The version string is cosmetic.** Show it in banners/logs; never
gate on it (`==`, `!=`, `<`). Verify the `service` field too, so you
fail loudly when the probe lands on the wrong port (an nginx/WordPress
on `:80` instead of the API on its real port is a classic footgun).

**4. One verdict function; everyone calls it.** Compatibility is decided
in exactly one place. A second ad-hoc check elsewhere (a post-deploy
smoke that re-rolls its own `version != EXPECTED`) will contradict the
first and reintroduce the drift.

The generalisable rule: **cross-component compatibility is `required
features ⊆ advertised features`, decided once; the version is a label,
not a gate.** Pair it with the §12.1 stamp — stamp says *which build*,
features say *what it can do*. See anti-pattern #41.

<a id="deploy-idempotent"></a>
### 12.3 Version-gate the deploy — skip the rebuild only on a stamp that catches every edit

Cross-building and shipping to a target on **every** onboard / deploy
is wasteful when the target already runs the exact code you'd build —
a cross-compile plus pushing a large bundle over a thin link, repeated
for no change. Make the deploy **idempotent and version-gated**: read
the target's reported stamp ([§12.1](#build-version-stamp)), compare it
to the source revision you're about to ship, and **skip the build +
ship when they're equal**. (Same idea as a [finish-the-operation skip
check, §18.7](#finish-the-operation), applied to a build.)

The gate is only as trustworthy as the stamp's sensitivity to change —
and this is where it silently breaks:

- **Gate on `sha + dirty`, never the bare commit sha.** If the version
  is just `git rev-parse HEAD`, an **uncommitted** edit (staged or in
  the working tree) carries the *same* sha as the last commit, so the
  gate reads "target already current" and **your new edit never
  ships** — the change you're actively testing is the one silently
  dropped. The gating identity must include a dirty bit.
- **Dirty must catch *every* uncommitted change.** Compute it with
  `git status --porcelain` (staged + unstaged + untracked), not
  `git diff` (unstaged tracked only) — the §12.1 trap. A half-blind
  dirty check is worse than none here: it makes the skip *look* safe
  while stranding staged work.
- **A `-dirty` stamp is always "different" — never skip it.** It's
  unreproducible ([§12.1](#build-version-stamp)), so it can't match a
  committed target revision; always rebuild + ship a dirty tree (or
  refuse it on a release path).
- **Confirm by reading the stamp back *from the target*.** "Skipped,
  already current" must mean the target *reports* that revision, not
  that the deploy script assumed it ([§28.3](engineering-rules.md#lib-version-stamp)'s
  verify-what-actually-landed) — a skip keyed on a stale local belief
  re-creates the gap the stamp was meant to close.

The generalisable rule: **make an expensive rebuild/redeploy idempotent
by version-gating it — skip when the target already reports the exact
revision you'd ship — but compute that revision as `sha + dirty` with
dirty = `git status --porcelain` (staged + untracked, not just
unstaged), or the gate hands an uncommitted edit the committed sha and
silently strands it; a `-dirty` stamp never matches, and confirm the
skip against the version the target actually reports.** See anti-pattern
#110.

---

<a id="source-vs-deployed"></a>
## 13. Source tree vs. deployed artifact

> **When this fires** — producing build artifacts, consuming a shared library, or running a path on a compiled / installed rig.
>
> **Do** — gitignore generated output; depend on a **published version** (never a sibling source tree); don't assume the dev layout at runtime; a hot-edit on the deployed box is a *loan* — capture it back same-day.

Keep the **source tree you build in** separate from the
**artifact others consume**. They are different things in
different places, and conflating them is how "I edited the
library but nothing changed" and "the build tree got committed"
both happen.

Two halves of one rule:

**1. Build artifacts are not source.** Compiled output —
`dist/`, `target/`, `build/`, `*.o`, `*.os`, `*.so` / `.dylib`
/ `.a`, wheels, `__pycache__`, `.venv` — is generated, not
authored. Gitignore it; never commit it into the source tree;
never hand-edit it. The source tree holds the inputs to the
build, not its outputs. (This is the same hygiene the embedded
rsync excludes enforce — §12 — applied to the repo itself.)

**2. Consumers depend on a published artifact, not your working
tree.** A library has a maintainer-facing **source repo** and a
separate **deployed/published form** that consumers point at — a
versioned wheel, a tagged tarball, a shared-prefix clone, a
package-registry entry. Consumers pin a version/tag; they do not
`import` from, build against, or symlink into the maintainer's
live source checkout.

```
   maintainer source repo                consumer project
   ┌────────────────────┐   publish      ┌──────────────────┐
   │ src/ … (authored)  │ ──────────────▶│ depends on        │
   │ build/ dist/ (gen, │   tag→package  │   lib @ vX.Y      │
   │   gitignored)      │   →prefix/     │   (pinned)        │
   └────────────────────┘    registry    └──────────────────┘
```

Why the separation is load-bearing:

- **Editing source doesn't silently change consumers.** The
  publish step is the deliberate boundary: build → package →
  publish to the prefix/registry → consumer bumps the pinned
  version. Without it, every consumer floats on the maintainer's
  uncommitted work-in-progress and breaks unpredictably.
- **Version pinning gives per-consumer control.** Each consumer
  adopts a new library version when *it* is ready (bump the
  `tag = "vX.Y"` / version constraint), the same opt-in model as
  any vendored dependency.
- **A separate deploy location keeps the build tree disposable.**
  You can `rm -rf` and rebuild the source tree's `build/` without
  touching anything a consumer relies on.

When you maintain a shared library:

1. Authored source lives in the source repo; generated output is
   gitignored.
2. Releasing is an explicit step (`tag → package → publish to the
   prefix/registry`), never "consumers read my `build/`."
3. Consumers depend on the published form by version, and bump
   deliberately.

If you catch a consumer importing from another project's `build/`
or `target/` directory, or pointing a dependency at a live source
checkout instead of a published version, that's the bug — see
anti-pattern #27.

<a id="dev-vs-deployed-layout"></a>
### 13.1 Don't assume the dev layout at runtime — the compiled / installed deployment differs

The source checkout you develop on and the **compiled / installed**
form you deploy have different layouts *and* different
capabilities. Runtime code that assumes one breaks on the other —
and the half you didn't run **passes silently**, so the breakage
surfaces only on the rig (and often takes *existing* features down,
not just new code). Two shapes recur:

- **No source / no `__main__` in a compiled package.** A package
  shipped Cython/`.so`-compiled (or frozen) has no `.py` and no
  module code object, so `python -m pkg.mod` fails on the rig
  ("No code object available") though it runs fine from source.
  **Import the module and call its entry function** instead
  (`python -c "import pkg.mod; pkg.mod._main()"` — importing the
  `.so` works, and argv is still parsed). Don't invoke a deployed
  module with `-m`.
- **Path resolution that bakes in the source-tree shape.** A
  `__file__`-relative / `parent.parent/…` path to a sibling
  `scripts/`, asset, or config is computed for *one* layout and
  wrong in the other — the installed package dir is not the repo
  root, and the frozen bundle is neither. Resolve against the
  **actual** deployed location, and where dev and deployed differ,
  **try both candidate layouts** (repo-root *and* installed/bundled)
  rather than the one your machine happens to have.

The discipline: a path or invocation that works from your checkout
is **unverified** until it runs against the deployed form. Exercise
the deploy/calibrate/launch paths on a **production-built** target
(compiled rig / frozen bundle), not just a source checkout — the
two diverge silently, and the rig is where it bites.

The generalisable rule: **the deployed artifact has a different
layout and fewer capabilities than your source tree — don't run a
deployed module with `python -m`, don't hard-code source-tree
paths, and verify every deploy/launch path on the
compiled/installed form, because the side you didn't test passes
silently and fails on the rig.** See anti-pattern #53.

<a id="depend-on-prefix"></a>
### 13.2 Depend on the published lib prefix, not a sibling source checkout

§13 in the concrete, for our shared libraries: a project that uses
`unomsg` (or any team lib) depends on the **compiled `unolib`
prefix** — git-dep the bare repo it publishes
(`unomsg = { git = "file://~/unolib/libs/unomsg.git", tag = "vX.Y" }`),
the Python wheel under `unolib/wheels/`, or the `lib/<triple>/` +
`include/` it ships — **never** a path-dep into the maintainer's
live source tree (`path = "../unomsg-rs/crates/…"`). Read the
upstream source all you like for reference; just don't *build*
against it.

Why this specific form, beyond §13's "don't float on WIP":

- **The prefix is the validated artifact.** It carries the
  per-target compiled libs the rigs actually run, the matching C
  ABI / wheel, and the rpath wiring (a binary consumer bakes
  `prefix/lib/<triple>` into its own rpath via a tiny `build.rs` —
  a dependency's link-args don't propagate). A sibling source
  checkout has none of that pinned and may have drifted from the
  last release.
- **The publish step is the boundary.** Upstream changes reach you
  only when the lib is rebuilt, republished to the prefix, and you
  bump the pinned tag — the same deliberate opt-in §13 wants.
- **A new wire type lands upstream, then flows down.** Need a new
  message/struct from the shared schema? Add it upstream, publish,
  bump — don't reach sideways into its source. And if the published
  client is *generic* over consumer-supplied schema types, the type
  can simply live in **your** repo: define it there and serialise it
  yourself rather than coupling to upstream internals.

This is the default posture for **future refactoring onto a shared
lib**: the moment a stack starts consuming `unomsg`/`unolib`, wire
it to the published prefix, not the dev checkout.

The generalisable rule: **consume a shared library through its
published prefix (git-dep the bare repo, the wheel, the
`lib/`+`include/` it ships) and pin a tag — never `path`-dep or
symlink into the maintainer's source tree; read that source for
reference only.** See anti-pattern #27.

<a id="capture-live-edits"></a>
### 13.3 A hot-edit on the deployed target is a loan from the repo — capture it back, and diff before you deploy

The drift between source control and a deployed box runs in **both
directions**. The familiar one is the target *behind* the repo (a rig
that never got the new build — diff the build stamps,
[§12.1](#build-version-stamp), anti-pattern #29). The quieter one is
the target **ahead** of the repo: a hot-fix applied directly on the
live box — a `.py` swapped over a compiled module to recover a rig, a
service file edited in place on the server, a feature grown on the
deployed copy because that's where the iteration loop was fast. It
*works*, so nothing forces the follow-through — and now the **repo is
stale** and doesn't know it.

Two failure modes follow, both shipped:

- **The next deploy clobbers the live improvement.** A reinstall /
  rebuild restores the committed state, silently reverting the hot-fix
  — the bug you fixed on the box comes back, looking like a
  regression (a live `.py`-over-`.so` swap undone by the next
  rebuild; an in-place server edit erased by the next sync).
- **The next deploy is diffed against the wrong baseline.** Whoever
  deploys next assumes the repo matches the box; the unexpected diff
  is discovered mid-deploy, or never.

The discipline:

- **A hot-edit is a *loan*, not a fix.** It buys recovery time
  ([§15.7](#no-reflexive-reboot) — narrowest live action), but the
  durable fix is the **committed source**; capture the edit back to
  the repo (same day, same session) or it *will* be erased. Until
  it's committed, the box is running code that exists nowhere else.
- **Before deploying to a long-lived target, diff the target against
  the repo** — not to trust the box, but to *catch* drift-ahead
  before you overwrite it. An unexpected difference is either a
  hot-fix to capture or a tamper to investigate; both beat a silent
  clobber.
- **Leave a marker on the box** when you hot-edit (the
  `.bak`-alongside convention, a log line): the next person diffing
  the target can tell a deliberate loan from corruption.

The generalisable rule: **deployed-vs-repo drift runs both ways — a
target behind the repo gets the missing build (#29), but a target
*ahead* of the repo holds work that exists nowhere else; capture a
live hot-edit back to source the same day, and diff the target before
deploying so a quiet live improvement isn't clobbered by the next
release.** See anti-pattern #80.

<a id="regenerated-location"></a>
### 13.4 An artifact in a build-regenerated location is ephemeral — install deps through the build, deploy paired halves together

A build or provision step **regenerates** parts of the tree — recreates
the venv, wipes `target/`, repaves the install prefix. Anything
**hand-placed inside a regenerated location** (a wheel `pip install`ed
into the venv's `site-packages`, a file dropped in `target/`, a tweak in
a cleaned cache) is **erased by the next build** — silently, because the
build "succeeded." The durable home for a runtime dependency is the
**build's own input** (the dependency manifest / lockfile it installs
from, [§21](#python-env)), not the regenerated output.

This bites hardest as a **half-reverting paired deploy**: you ship a
change in two parts — code in a **durable** spot (outside the rebuilt
tree) and its new dependency **inside** the regenerated one — and they
diverge on the next build. The code survives, the dep vanishes, and the
system comes up running the new code against the missing dep: not a
crash, a **silent degrade** to a fallback path. We shipped this twice —
a hand-installed wheel in the venv (the next onboard recreated the venv;
the matching code lived outside it and survived, so the feature ran on
fixed defaults), and a calibration left in `site-packages` instead of
its durable store.

- **Install runtime deps through the build, not by hand into the
  regenerated tree.** Add the wheel to the lockfile / bootstrap install
  phase ([§21](#python-env)) so every rebuild reinstalls it. A by-hand
  `pip install` is a [loan (§13.3)](#capture-live-edits) — capture it
  into the build's inputs the same day.
- **Deploy paired halves through the same repaved path.** If a code
  change needs a new dependency, ship both via the build — never one half
  durable + one half hand-dropped where the build will wipe it.
- **Verify on the rebuilt form, by *loading* — not "the file is
  there."** Run the rebuild, then confirm the consumer actually
  imports/loads the dep ([§11.2](#freeze-native-deps),
  [§13.1](#dev-vs-deployed-layout)); a CRITICAL-on-missing log beats a
  silent fallback.

The generalisable rule: **whatever a build or provision step regenerates
is ephemeral storage — a dependency or artifact hand-placed there is
wiped by the next build; install runtime deps through the build's own
manifest so every rebuild restores them, ship a code-plus-dependency
change as one paired unit through the repaved path (never a durable half
+ a wiped half), and verify by loading on the rebuilt form.** See
anti-pattern #95.

<a id="vendor-not-submodule"></a>
### 13.5 Share must-be-present content as vendored copies, not a submodule

When a set of files must simply *be present* in every consumer repo —
shared coding rules, lint / CI config, a license header, a `.proto` or
schema everyone reads, doc templates — a **git submodule is the wrong
tool.** It adds a fetch / init / transport step that breaks for reasons
that have nothing to do with the content:

- a clone without `--recurse-submodules` leaves **empty dirs** (and the
  consumer doesn't notice until something reads them);
- a submodule URL that's a `file://` / local-mirror path, or reached via
  a `url.…insteadOf` rewrite, is **blocked by git 2.38+**
  (`fatal: transport 'file' not allowed`, CVE-2022-39253) — so **every
  build that recurses submodules fails**;
- a private / unreachable URL fails in CI or on a fresh box.

The content has no independent build or version lifecycle, so that
fragility buys nothing.

- **Vendor committed copies.** Commit the files directly into each
  consumer; a small **copy-in step** (a script that copies from the
  source repo, **path-limited** so WIP is untouched, and commits only
  when something changed) propagates updates. A fresh clone has the
  content with **zero setup** — nothing to fetch, init, or reach.
- **The cost is a propagation step, not a dependency.** You trade "one
  source of truth, auto-followed" for "copy-and-commit on change" —
  worth it for static content; **not** worth it for versioned **code**,
  which *does* have a lifecycle and belongs behind a published prefix
  ([§13.2](#depend-on-prefix)).
- **If you must use a submodule**, its URL has to be reachable from
  **every** consumer (a real remote, never `file://` / a local mirror)
  and consumers must actually init it — otherwise you've shipped empty
  dirs.

The generalisable rule: **content that must merely be present in every
repo (rules, config, schemas, templates) is vendored as committed copies
with a path-limited copy-in propagation step — not a git submodule, which
breaks builds on a missing `--recurse-submodules`, a `file://` /
local-mirror URL (blocked by git 2.38+, CVE-2022-39253), or an
unreachable URL; reserve submodules / prefix-deps for versioned code
([§13.2](#depend-on-prefix)).** See anti-pattern #122.

---

<a id="bundled-config"></a>
## 14. Bundled configuration and shipping defaults

> **When this fires** — shipping configuration inside a frozen / deployed bundle.
>
> **Do** — bake explicitly (never automatically), gitignore the output, and have the runtime **seed from it, not trust it**.

A frozen / deployed bundle ("the .app", "the wheel", "the
installer") often needs to ship **with the operator's working
config already in it** — VPS / fleet-registry URL + token, per-
target SSH aliases, default cloud endpoints. A fresh laptop
opens the bundle and is immediately useful instead of staring
at an empty Settings dialog.

The pattern:

1. A `scripts/bake-config.py` (or equivalent) **snapshots the
   operator's local config** to a bundle-root file
   (`<project>_config.json`).
2. The build step bundles that file into the deployed artefact.
3. The runtime, on first launch, checks for the bundled file —
   if present, it seeds the live config from it; if absent, the
   normal first-run flow takes over.
4. The runtime **never writes** to the bundled file. Live config
   changes go to the user-home config; the bundle is read-only.

<a id="bake-explicit"></a>
### 14.1 Bake is explicit, never automatic

`bake-config.py` runs by hand. The GUI itself, the test suite,
`build.sh` — none of them invoke it implicitly. The operator
explicitly chooses to capture "my current state" into the next
build.

Why explicit: an automatic bake at build time would silently ship
whatever the laptop happened to have configured, including stale
URLs, half-finished tokens, and any credential the operator
imported and then disowned. Make the capture a deliberate act.

<a id="bake-gitignored"></a>
### 14.2 The output is gitignored

The bundled file contains a bearer token. **Add it to
`.gitignore` the moment you wire up the bake** — committing it
once into a public repo is a token-rotation event. The bake
script itself should add the entry idempotently the first time
it runs:

```python
def _ensure_gitignored(repo_root: Path, name: str) -> None:
    gi = repo_root / ".gitignore"
    lines = gi.read_text().splitlines() if gi.exists() else []
    if name not in lines:
        gi.write_text("\n".join(lines + [name, ""]))
```

### 14.3 Distribute through trusted channels only

A bundle that contains a fleet token is **as sensitive as the
token**. Don't:

- Publish it to a public release page.
- Drop it in a shared cloud bucket without per-recipient ACLs.
- Email it. (Email is a public bucket with a slow leak.)

Do:

- Treat the bundle as a credentialed artefact. Same distribution
  channels you'd use for SSH keys or API tokens.
- Rotate the embedded token if a bundle leaks or a recipient
  leaves the operator pool.

### 14.4 Runtime reads the bundle, doesn't trust it

The runtime's config loader reads the bundled file as a **seed**,
not as a sealed source. Schema validation, version checks, and
field-by-field rebinding happen on first load just as they would
for a user-home config. A field the runtime doesn't recognise is
dropped with a WARNING; a field with a now-invalid value is
overridden with the runtime's default. Defence in depth — the
bundle shouldn't be able to brick a fresh launch.

<a id="secret-free-bundle"></a>
### 14.5 Prefer a secret-free bundle — ship non-secret defaults, seed the credential at runtime

§14.2–14.3 *manage* a bundle that contains a secret (gitignore it,
treat it as a credential). Better: **don't put the secret in the
bundle at all.** A frozen / deployed artifact (a bundle, image,
installer, wheel) is **distributable and extractable** — every
recipient, and anyone it leaks to, gets a byte-for-byte copy of
whatever's baked in. A baked bearer token / API key / shared
password is therefore a **fleet-wide secret** that leaks to the
whole recipient set at once, and any single leak rotates it for
**everyone**. The convenience trap is snapshotting the builder's
personal `~/.config` "so it's ready to use" — that embeds *their*
credentials into the artifact.

- **Bake only non-secret defaults.** Ship a **checked-in**,
  secret-free default (service URLs, non-secret targets, the SSH
  alias, feature toggles) so a fresh install is useful out of the
  box with **zero** credentials. The default lives in the repo
  because it's not sensitive — it is *not* snapshotted from a
  configured laptop.
- **Seed the credential per-device at runtime**, never at bake — the
  operator's own SSH key, a provisioning step, an env / keychain
  entry, a tokenless audited enrollment
  ([§26.4](#per-identity-access)). The artifact
  *acquires* a per-identity credential at first run; it doesn't
  *carry* a shared one.
- **Gate the ship on "no secret present."** A bake / build step that
  greps the artifact for any secret field (token / key / password)
  and **refuses to package** if one is found — defence in depth, so a
  future careless bake can't silently reintroduce the leak. That grep
  is the tripwire the [§14.1](#bake-explicit) explicit-bake discipline
  leans on.

The payoff: the artifact carries **zero secrets**, so it's no longer
a credentialed artefact ([§14.3](#bundled-config)) — it ships through
normal channels and a leak is a non-event.

The generalisable rule: **a shippable / extractable artifact carries
no secret — bake only non-secret defaults, seed the per-device
credential at runtime (a per-identity key / provisioning / tokenless
enrollment), and gate the ship on a grep that refuses to package any
secret field; "the bundle is sensitive, handle it carefully" is the
fallback, "the bundle has zero secrets" is the design.** See
anti-pattern #113.

---

<a id="diagnostics"></a>
## 15. Diagnostics

> **When this fires** — chasing a bug, about to reboot to clear a symptom, or about to call something a performance 'floor.'
>
> **Do** — **diagnostic-first** — produce the command that proves the cause before you change code; capture volatile state before any reset; name the mechanism and the alternatives ruled out.

<a id="diagnostic-first"></a>
### 15.1 Diagnostic-first, fix-second

Before you change code, **produce the command that proves the
cause**. The shape that recurs:

> "Operator reports symptom X."
> → first artifact is a one-liner that counts / decodes / reads
>   the kernel/proc/journal state that distinguishes the
>   hypothesised causes.
> → the one-liner's output decides which fix to apply.
> → the one-liner gets pinned somewhere it survives (this rule
>   set, a probe script, a memory file).

Examples of the shape:

```sh
# "video stuck / lose signal after a while"
journalctl -u <service> --since "60s ago" | grep -c "<per-tick phrase>"
# >20 = log-spam regression; ≤1 = something else

# "topic frozen after operator logout"
ls -i /dev/shm/<topic-shm-path>     # compare inode pre/post logout

# "session superseded by new connection every 10–12 s"
pgrep -af <project root>            # find orphan workers BEFORE
                                    # blaming the server side

# "default ON feature is off on fresh install"
sudo cat /var/lib/<paramsdir>/UseFeature 2>/dev/null
# returns nothing → unset → confirms [§7.1](engineering-rules.md#params-trap) trap
```

Without the diagnostic, the fix is guesswork. With it, the fix
is forced. **The diagnostic that proves the cause is more
valuable than the fix itself** — it survives across sessions; the
fix is one-shot.

### 15.2 Probes accumulate; new probes start from zero

Prefer extending an existing probe / diagnostic script over
inventing a new one. Probes accumulate the right exit codes, ANSI
formatting, edge cases, and timeouts over months; a brand-new
probe starts from zero on all four. If you need to add a new kind
of check, see if the existing probe already has a section that
maps to it.

### 15.3 Hand-runnable on the target

Diagnostics that need access to a remote target should be
self-contained scripts hand-runnable in a shell, not Python
modules that only work under the framework. A probe you can `scp`
and `python -u` on the target is debuggable on day one.

### 15.4 Empirical thresholds are documented in place

If a probe decides "broken" vs "OK" by a numeric threshold (a
brightness std, a fps floor, a journal-line count), document the
threshold and where it came from at the call site:

```python
# np.std(decoded_y) < 10 ≡ gray.
# Broken case measured std≈4.6; sensor-noise floor ≈10; real
# outdoor content ≥30. Threshold 8 picked to bracket noise.
if np.std(y) < 8.0:
    return "gray"
```

A bare `< 8.0` will be wrong six months later when nobody
remembers why.

<a id="link-vs-data"></a>
### 15.5 "Link up" ≠ "data flowing" — check both layers

A transport can be **healthy at the link layer and silent at the
application layer** at the same time: the bus / socket / channel
is up, error-free, correctly configured — and zero useful frames
are crossing it, because the *producers* on the far end aren't
transmitting. Checking only the link declares "fine" while the
feature is dead.

The shape (a CAN bus is the canonical case, but it generalises to
any link with a separate producer):

```sh
# Layer 1 — link health: is the interface up and error-free?
ip -details -statistics link show can1   # ERROR-ACTIVE, 0 bus-errors → link OK
# Layer 2 — is data actually flowing?
timeout 5 candump can1 | head            # 0 frames → producers silent, NOT "fine"
```

Link UP + 0 bus-errors + 0 frames means the wire is healthy and
**nothing is talking on it** — power/wiring to the sensors, a
missing enable/wakeup TX the devices need before they stream, or
the producers simply not running. "The bus is fine" is a
layer-1 statement; the operator's "it's not working" is a
layer-2 problem.

Two rules fall out:

- **A liveness probe asserts on frames received, not link
  state.** "Healthy" requires data crossing, not just a
  configured interface. Report both layers separately so the
  diagnosis is unambiguous: `link=UP frames=0` localises the
  fault to the producer side in one line.
- **Don't fill in expected IDs / addresses as placeholders and
  treat them as known.** Until a real on-target scan shows the
  live IDs, a `RADARS = [0x000, …]` placeholder is *unverified* —
  mark it so, and make discovery (`--scan`) the thing that
  populates it. A placeholder mistaken for ground truth is its
  own silent bug.

Same family as anti-pattern #17 (daemon "runs" but publishes
nothing): there the silence is producer-side in *our* process;
here it's producer-side across a *transport*. Both are caught
only by checking the data layer, never the "is it up" layer. See
anti-pattern #32.

<a id="optimization-floor"></a>
### 15.6 A "hard floor" / "can't optimize this" claim names its mechanism and the alternatives it ruled out

"This is as fast as it gets," "that's a hardware floor," "X can't
be reduced" is a conclusion that **stops investigation** — so it
has to earn the stop. The same shape has been wrong three ways:

- A data-ingest cost was called a fundamental **hardware floor** —
  it was an artifact of the *wrong pipeline element*. Swapping one
  element (a zero-copy capture source for a copy-through one) cut
  the cost ~7× *and* a downstream latency-critical consumer got
  faster. The floor was the tool, not the silicon.
- "Lower the input rate to save CPU" did nothing, because the cost
  is paid at the *dequeue*, **before** the drop point — a genuine
  sub-floor, but only for that one lever; it never bounded the
  whole.
- "This accelerator stage is 220 ms, too slow" was a framework
  **re-tracing its graph every call**, not device time
  ([§9.7 r1](engineering-rules.md#cpu-gpu-decision)); compiled/cached, the same path
  was ~14 ms.

A floor claim is load-bearing only if it names **(a) the mechanism
that makes it fundamental** — the physics, the fixed-function unit,
the per-item dequeue copy; *not* "it's just slow" — and **(b) the
alternative paths it ruled out** — the other element, the other
backend, the zero-copy route. Watch the **superset assumption** in
particular: "the general engine does everything the fixed-function
unit does" is often false (their supported input formats differ
*both* ways), and it's exactly how a *tool* limit gets mistaken for
a *hardware* floor. Before you write "can't be improved," attribute
the cost to its mechanism ([§9.7 r1](engineering-rules.md#cpu-gpu-decision)) and try the
one obvious alternative path you haven't.

The generalisable rule: **"it can't go faster" must cite its
mechanism and the alternatives it ruled out — an unattributed floor
is usually the wrong tool or a misattributed cost, and it costs
more than the slow path because it stops anyone from looking.** See
anti-pattern #46.

<a id="no-reflexive-reboot"></a>
### 15.7 Reboot is a last-resort recovery, not a debugging step — capture volatile state first

Rebooting the box (or power-cycling a device, or even bouncing a
service) to make a symptom disappear is the most seductive debugging
bad habit: it *works often enough* to feel like progress, and it
**destroys the evidence every single time.** A reboot is a recovery
action, not a diagnostic one — don't reach for it first.

What a reboot throws away is exactly the state that would localise
the cause:

- the kernel ring and journal since boot (`dmesg`, logs not yet
  flushed to disk),
- the stuck thread's `wchan` / blocked syscall,
  `/proc/<pid>/{status,maps,fd}`,
- the wedged device's current registers / link state / queue depth,
- the process tree — who held what, the orphan that was still alive.

And it converts a **reproducible** fault into an **intermittent
ghost**: the symptom is gone, no cause is named, and it returns at
the worst time. "A reboot fixed it" is an *open* bug, not a closed
one.

The discipline:

- **Capture before you reset.** Before any destructive recovery,
  dump what you'd need to diagnose to a file that survives the
  reboot — the journal / `dmesg`, `/proc/<pid>/wchan` + status, the
  device state, a process list. Five seconds of capture beats
  re-debugging from zero after the state is gone.
- **Escalate narrowest-first.** Re-open the one handle → reset the
  one device → restart the one daemon → **re-initialize the wedged
  *subsystem's driver*** (unbind/rebind it, re-train the link) →
  *then*, only if a captured diagnosis shows the fault is genuinely
  not software-recoverable (the wedged-firmware case,
  [§6.2 rule 2](engineering-rules.md#restart-policy)), a full reboot. Don't skip to the
  biggest hammer — and note that **a reboot is not always the
  biggest hammer**: a *warm* reboot doesn't power-cycle a
  co-processor / SerDes / FPGA, so a peripheral wedged below the
  main CPU can survive it. Re-initializing that subsystem's driver
  (its probe re-runs the device's init sequence) is **both narrower
  *and* more effective** — one rig cleared a wedged camera
  deserializer by unbind/rebinding its sensor driver after two warm
  reboots had failed.
- **Reboot deliberately, and record why.** When a reboot *is* the
  right call, note what you observed that justified it — which
  probe, which 0-Hz topic, which failed software-reset — so "needed
  a reboot" is a data point with a mechanism
  ([§15.6](#optimization-floor)), not a reflex that hides a
  recurring bug.

The generalisable rule: **a reboot/restart that makes the symptom
vanish without a named cause has destroyed the evidence and closed
nothing — capture volatile state first, recover with the narrowest
action that works, and treat a power-cycle as the deliberate last
resort [§6.2](engineering-rules.md#restart-policy) describes, never the first thing you
try.** This is [diagnostic-first (§15.1)](#diagnostic-first) applied
to recovery actions, not just code edits. See anti-pattern #47.

<a id="hypothesis-discipline"></a>
### 15.8 Converge by controlled comparison — diff the working reference, one hypothesis at a time, and three failed fixes means stop

Once [§15.1](#diagnostic-first) has the symptom reproduced, *how* you
converge on the cause matters. Three moves keep the search systematic
instead of a patch-spiral:

- **Diff against the nearest working reference.** Almost always,
  something *almost identical* already works — the sibling camera
  that streams, the other daemon that subscribes fine, the same
  pipeline on the other rig, the previous commit. Read the working
  one **completely** (not the half you remember) and **enumerate every
  difference** — config, env, versions, call order, defaults —
  then bisect the differences. "What's different from the one that
  works?" localises faster than reasoning about the broken one from
  first principles, and it's the debugging twin of the build-stamp
  diff ([§12.1](#build-version-stamp)).
- **One hypothesis, one variable, smallest change.** Write the
  hypothesis down, make the *minimum* change that tests it, and read
  the result. If it's wrong, **revert and form a new hypothesis** —
  never leave the failed fix in and stack another on top. Stacked
  speculative fixes destroy the comparison (you can no longer tell
  which change did what) and ship dead "just in case" code; this is
  how the blind-revert spiral starts
  ([#49](anti-patterns.md#recurring-shapes)).
- **Three failed fixes = stop fixing.** If three distinct,
  evidence-based attempts haven't closed it, the problem is almost
  never the line you keep editing — it's a level up: a wrong
  assumption, a design flaw, the wrong component blamed. Stop
  patching, write down what the three attempts *proved impossible*,
  and take it to a design discussion ([§24](#when-in-doubt) — ask) or
  a recorded decision ([§18.3](#decision-record)). A fourth patch on
  a misdiagnosis just buries the evidence deeper.

The generalisable rule: **converge on a cause by controlled
comparison — diff the broken thing against the nearest working
reference and test one written hypothesis at a time with the smallest
change, reverting failures instead of stacking them; and when three
evidence-based fixes have failed, the bug is a level up — stop
patching and re-examine the design.** (From the systematic-debugging
discipline in [obra/superpowers](https://github.com/obra/superpowers),
adapted.)

<a id="rate-meter"></a>
### 15.9 Measure a rate by counting events in a window — smoothed inter-arrival intervals lie on bursts

A rate readout (a topic-Hz meter, a frames-per-second badge, a
throughput probe) computed as a **smoothed reciprocal of inter-arrival
time** (EWMA of `1/Δt`) is an instrument that lies precisely when
someone is looking: any **burst** — a backlog draining after a stall,
a coalesced batch delivery, a scheduler hiccup — produces a handful of
near-zero `Δt`s, and the meter screams an absurd rate (a "4 000 Hz"
reading on a ~50 Hz topic) long after one smoothing constant's worth
of decay. The operator sees a number no real source could produce and
starts debugging a fault that doesn't exist — the meter measured its
*own* artifact, not the stream
([looks-healthy ≠ working, #2](anti-patterns.md#recurring-shapes) in
reverse: looks-broken ≠ broken).

- **Count events in a fixed window** (`N events / T seconds`, sliding
  or tumbling). A windowed count is burst-robust by construction: a
  drained backlog raises the count by exactly the drained messages,
  bounded and interpretable — never a reciprocal blow-up.
- **An impossible reading indicts the meter first.** A rate far above
  what the producer can physically emit is a measurement artifact
  until proven otherwise — check the meter's math before the stream
  ([§15.6](#optimization-floor)'s attribution discipline, applied to
  instrumentation).
- Same family as the shared probe that false-flags an encode-only
  source ([§10](engineering-rules.md#per-stream-isolation)): **a diagnostic that misleads
  under edge conditions is worse than no diagnostic** — operators
  either panic at ghosts or learn to ignore it
  ([§16.6](#logging)).

The generalisable rule: **compute a displayed or alerted rate by
counting events over a window, never by smoothing reciprocal
inter-arrival intervals — a burst makes `1/Δt` scream an impossible
number and the operator debugs a ghost; an impossible reading indicts
the meter before the stream.** See anti-pattern #82.

<a id="liveness-from-data"></a>
### 15.10 Derive liveness from the data link, not a side-channel probe

[§15.5](#link-vs-data) is "link up ≠ data flowing" on one channel.
The cousin spans two channels: a status indicator that asks a
**separate, out-of-band probe** "is it up?" — a fresh SSH handshake,
an HTTP healthcheck poll, a liveness ping — *while the system is
demonstrably up*, streaming its real data on a different connection.
The badge reads **down** while the payload flows.

Why the probe lies: the data tunnel is an **actively-streamed,
persistent** connection that stays warm; the probe's connection sits
**idle between polls**, so a shared edge drops it
([§23.3](#edge-new-conn-throttle)) and re-handshaking is throttled —
the probe false-reads "unreachable" forever. Now there are **two
sources of truth** for one question and the flaky one wins the badge.

The discipline:

- **The producer publishes its own state as telemetry on the primary
  data link** — a first-class status message (a supervisor's
  supervised-process set, a daemon's `live: bool` + last-update
  stamp) flowing on the *same* channel as everything else, at a
  modest rate, best-effort (publishing status never stalls the real
  work). Status is a topic, not a question you ask out-of-band.
- **The consumer derives the up/down verdict from telemetry
  liveness.** If the payload is flowing, the source is up *by
  construction* — never let a failed side-channel probe report
  "down" over a live data link. A probe may only **enrich detail**
  (a per-daemon breakdown) when it happens to connect; it is never
  the up/down authority.
- **Hold the verdict steady** — a status that flips per poll chases
  the flaky probe; apply hysteresis
  ([§3.6](engineering-rules.md#decision-hysteresis)) so a single
  missed sample doesn't flicker the badge.

The generalisable rule: **read liveness/health from the primary data
channel the producer already streams on — have the producer publish
its own state as telemetry there, and derive up/down from that. A
separate out-of-band probe (a fresh SSH/HTTP handshake per poll) sits
idle, gets dropped/throttled by the edge, and false-reads "down"
while the data flows; let it enrich detail at most, never override a
live link, and hold the verdict with hysteresis.** See anti-pattern
#107.

---

<a id="logging"></a>
## 16. Logging discipline

> **When this fires** — logging inside a loop / hot path, or on an embedded kernel console / serial UART.
>
> **Do** — transition + heartbeat, **never per-tick**; rate-limit per source — a slow sink pins a CPU core and can reboot the SoC.

Logs are an interface. Every line is information density vs.
display cost. Hot loops produce thousands of frames a second; an
unconditional `log("state = %s", state)` inside one of them is
not "verbose debugging" — it's a denial-of-service against the
journal and (worse) against asyncio loops that share the writer.

<a id="transition-heartbeat"></a>
### 16.1 Transition + heartbeat, never per-tick

A per-tick log line is almost always wrong. Replace it with two
emissions:

- a **transition** log on state change (the value differs from
  the last emitted value);
- a **heartbeat** every N seconds even when state hasn't changed,
  so liveness is still visible during long idle windows.

```python
_HEARTBEAT_S = 5.0
_last_emitted = (None, 0.0)

def log_mode(mode):
    last_mode, last_t = _last_emitted
    now = time.monotonic()
    if mode != last_mode or now - last_t > _HEARTBEAT_S:
        log.info("mode = %s", mode)
        _last_emitted = (mode, now)
```

Numbers from a real fix: a 100 Hz daemon that logged "manual
lock; idle" every 20 frames went from ~150 lines per 30 s to ~6.
The operator complaint that triggered the fix was
*"manual_lock pops up constantly"*. The bug was the log line.

### 16.2 Rate-limit per-source, not per-line

When a degraded condition produces the same warning many times
per second, rate-limit. Once per N seconds per source key is
typical:

```python
@rate_limited(per_second=0.2, key=lambda *, source: source)
def warn_uniform_frame(source: str, std: float):
    log.warning("%s: uniform-output skip (std=%.1f)", source, std)
```

The error is still visible; the journal isn't a flood.

<a id="log-spam-starves-loop"></a>
### 16.3 Per-tick log spam can starve the asyncio loop

The cascade we've shipped:

> Per-tick log line on N inactive subjects → asyncio writer
> contends → aiortc sends fall behind their buffered-amount cap
> → the **active** subjects degrade → operator sees "network
> flaky." It isn't the network. It's the log writer.

Treat any per-tick `log.*` call in an asyncio context as a
performance bug, not a debug aid. See also anti-pattern #19.

<a id="live-field-convention"></a>
### 16.4 Diagnostic flags are part of the contract

Each major service publishes a `live: bool` (see [§8.2 snapshot pattern](ui-rules.md#snapshot-pattern)) and a
last-update timestamp. The UI reads those instead of polling the
log. A subscriber going stale should be visible from a snapshot
field, not deducible only from absence of recent log lines.

<a id="log-levels"></a>
### 16.5 Log levels mean what they say

- `DEBUG` — only useful during active investigation; off in
  production.
- `INFO` — significant state transitions an operator might want
  to see in the journal (engage / disengage, transport switch,
  device connect).
- `WARNING` — degraded but recovering (retry pending, fallback
  taken, queue near cap). Loud enough to notice; not loud enough
  to page.
- `ERROR` — operator needs to take action.
- `CRITICAL` — the process is about to die / has died.

Don't emit at `ERROR` for a condition the code recovers from
silently. The level is a contract with the operator about
whether they need to do anything.

<a id="red-is-no-signal"></a>
### 16.6 A perpetually-red signal is no signal — fix it or remove it

A status signal — a CI check, a health badge, a dashboard light,
a monitor — only has value if **green means good and red means
act.** A check that is *always* red (a CI workflow that's been
failing for months, an alert that fires every boot, a "degraded"
badge that's never been green) trains everyone to ignore it. Then
the day it goes red *for a real reason*, nobody looks — the
signal is dead, and worse than absent, because it occupies the
slot where a working signal would be.

The recurring decision: a stale always-failing CI workflow is
**removed or fixed, never left red.** We deleted several
perpetually-failing GitHub-Actions workflows + a dead Jenkins
job across repos in one pass for exactly this reason — a red
that everyone has learned to scroll past is noise wearing the
costume of a signal.

The rule:

- **Every always-on signal is either green-in-the-good-state or
  deleted.** A check that can't be made to pass on the current
  codebase is removed until someone will own making it pass —
  not committed broken "to fix later."
- **A new check lands green.** Adding a CI job / monitor / badge
  that's red on day one just adds to the noise floor; make it
  pass first, or gate it off until it can.
- **Don't normalise red.** "Oh, that one's always failing,
  ignore it" is the failure state, not a workaround. The moment
  a team says that out loud, the check has negative value —
  remove it or fix it that day.

This is the macro form of [§16.5](#log-levels) (don't `ERROR` for a recovered
condition) and [§16.1](#transition-heartbeat) (transition + heartbeat, not constant
noise): a signal that's always asserting carries zero
information. See anti-pattern #34.

<a id="slow-log-sink"></a>
### 16.7 A slow log sink starves the system, not just the journal

[§16.3](#log-spam-starves-loop) is about a hot-path log starving an
*asyncio loop*. On an embedded target the same spam can starve a
**CPU core** and take down the whole box — because the *sink* is
slow. The kernel console is the classic trap: `console_loglevel`
routes `printk` to a serial UART, and a 115200-baud line drains a
few KB/s, so a few hundred `WARNING`/s back up in the `printk`
kthread and **peg a full core** feeding the UART. On a board with
isolated cores that's a core you can't spare — and the starvation
cascades: in one case the pinned core starved the GPU's CPU-side
power-management servicing, the GPU driver hit a PMU timeout, and
**software-rebooted the SoC** — a "GPU reboot" whose root cause was
a log line.

The discipline:

- **Don't let a slow sink gate the system.** Keep high-rate
  messages off the slow path: lower `console_loglevel` so
  `WARNING` spam never reaches the UART, while the messages still
  land in the in-RAM ring (`dmesg`) and journald (fast). Persist
  the setting where the platform reads it — a checked-in config /
  provisioning file, *not* an env var ([§7.5](engineering-rules.md#env-var-config)).
- **A high-rate source is a bug even when "expected."** A sensor
  emitting `discarding frame` ~150/s is benign at the data layer
  and lethal at the console layer; rate-limit it (§16.2) or raise
  its level so it isn't on the hot console path.
- **Attribute the cascade — don't blame the loudest subsystem.** A
  reboot that *names* a subsystem (a GPU PMU timeout) can be a
  victim of starvation elsewhere, and the newest code in that
  subsystem is the tempting-but-wrong suspect. Capture the evidence
  that survives the reboot (`/sys/fs/pstore/console-ramoops-*`,
  [§15.7](#no-reflexive-reboot)) and attribute the mechanism
  ([§15.6](#optimization-floor)) before reverting anything — a
  blind revert of the wrong component is the
  [decision-oscillation shape #6](#decision-record).

The generalisable rule: **logging cost is paid at the slowest
sink, not at the log call — on an embedded target a high-rate
message routed to a serial console can pin a CPU core and cascade
into a whole-system fault, so keep the hot path off the slow sink
(low `console_loglevel`, rate-limit the source) and attribute any
resulting fault by captured evidence, not by blaming the loudest
subsystem.** See anti-pattern #49.

<a id="format-log-once"></a>
### 16.8 Format each line once — one owner of the prefix

A logging framework's formatter already stamps every line with a
**timestamp, a level, and the target** — and the target *is* the
component identity (the crate/module/logger name). So when the
code *also* hand-prepends a `[component]` tag inside the message,
the identity lands on every line **twice**:

```
[2026-06-16T12:00:01Z INFO  selfdrived] [selfdrived] carState 50Hz
                      ^^^^   ^^^^^^^^^^  ^^^^^^^^^^^^
                      level  target      hand-prefix — duplicate
```

The level/timestamp double up the same way if a second layer
re-stamps. The shapes that produce a double-formatted line:

- **Framework prefix + manual prefix.** The formatter emits the
  target and the message hand-writes `[name] …` too. Pick **one
  owner**: either let the framework own ts/level/target and write
  *bare* messages, or turn the framework's target off
  (`.format_target(false)` / a format string without `%(name)s`)
  and own the tag in the message — never both.
- **Two init paths / divergent setups.** One binary calls the
  framework's bare `init()` (default format) while its sibling
  builds a custom `Builder`/`Formatter` — same fleet, two line
  formats, and a library that *also* installs a logger stacks a
  second formatter on the same record. Configure the format in
  **one shared init** every binary calls; a library logs through
  the facade and installs **no** formatter.
- **A supervisor re-logging a child's output.** A parent that
  captures a child's *already-formatted* stdout and re-emits it
  through its own logger stamps a second timestamp/prefix on each
  line. Forward captured child output **verbatim** (or let the
  child log directly to the shared sink) — don't re-format a line
  that's already a log line.
- **Re-interpolating a formatted string.** Running an
  already-rendered message back through the formatter (`%`/`{}`)
  a second time corrupts or panics on stray format tokens. Format
  the record once at emit; pass structured fields, not
  pre-rendered text, to the logger.

The generalisable rule: **a log line is timestamped, levelled, and
prefixed with its source exactly once — by a single owner. The
logging framework already stamps ts + level + target (the
component identity), so don't hand-prepend the same `[component]`
tag, don't stack a second formatter (a divergent per-binary init
or a library that installs its own), and don't re-log a child's
already-formatted output at a supervisor. One init, one format,
one prefix.** See anti-pattern #106.

<a id="one-record-format"></a>
### 16.9 One record format — the unomsg log; Rerun views it, never persists it

A line log (§16.1–16.8) is for humans tailing a console. The other
thing a system records is its **message streams** — the typed
pub/sub traffic, captured to disk for replay, diagnostics
([§15](#diagnostics)), upload, and offline analysis. Every uno
system records that **exactly one way: the unomsg log of record**
(`unomsg::ulog` segments — `service_id | mono_ns | len | payload`,
the payload being the raw wire bytes, rotated into
`unomsg-*.log`). It's a positive org convention like
[Rust](engineering-rules.md#rust-vs-python) (§9),
[`unolib`](engineering-rules.md#reuse-critical-path) (§28), and
[Rerun for *viewing*](ui-rules.md#rerun-visualization) (§8.30): a
new system **adopts** the shared record, it does not invent a
per-tool capture format.

- **One format, one shared module — fleet-wide.** A `loggerd`
  subscribes the `should_log` topics and writes the segment; a
  replay tool reads it. Both go through the **same**
  `unomsg::ulog` writer/reader, so the framing can't drift between
  producer and consumer ([§28](engineering-rules.md#reuse-critical-path)).
  Every uno stack — road, slam, whatever's next — writes the same
  `.log`, so one toolchain (replay, upload, the deleter, analysis)
  works across the whole fleet.
- **Rerun is a VIEW, never a record.** [Rerun](ui-rules.md#rerun-visualization)
  (§8.30) renders the data live or replayed — but it does **not**
  persist. Stream live with `connect_grpc()` to a running viewer,
  or replay the canonical `.log` into a viewer with a `ulog-rerun`
  decoder; **never `rr.save("…​.rrd")`.** An `.rrd` on disk — like
  a bespoke per-tool `jsonl`/CSV dump — is a **second log format
  beside the one of record**: the classic two-copies trap
  ([§28](engineering-rules.md#reuse-critical-path)) — the two drift,
  the wrong one drives, and a consumer reads the stale one. The
  `.log` is the single source of truth; Rerun is a transient lens
  on it.
- **One decoder, not two.** A replay (`.log` → Rerun) decodes each
  record with the **same** message readers the daemons and the GUI
  use — never a second, parallel decoder to keep in sync (§28).
  The replay binary is *just* the stack-specific `Event`→entity
  mapping over the shared engine.
- **One clock.** Every record carries `mono_ns`
  ([§2.1](engineering-rules.md#timestamp-contract)); the live view
  and a replayed `.log` land on the same `mono_time` timeline, so
  a recording and a live session line up.

The generalisable rule: **a uno system records its message streams
in exactly one format — the shared `unomsg::ulog` log of record,
written and read through the one `unomsg::ulog` module so it can't
drift. Rerun (§8.30) *views* that data — live via `connect_grpc`
or replayed from the `.log` — but never persists a parallel `.rrd`
(or any bespoke per-tool dump) beside the canonical log; two record
formats are the §28 two-copies trap. One record, one decoder, one
clock; Rerun is the lens, the `.log` is the truth.**

<a id="torn-record-recovery"></a>
### 16.10 A record-stream reader recovers from a torn trailing record

The writer of an append-only log / record stream (the §16.9 log of
record, a WAL, an event log, a framed socket, a chunked upload) can
be **killed mid-append** — power cut, OOM, `SIGKILL`, a full disk.
That leaves a **torn trailing record**: the length prefix is flushed
but the body is cut short. A reader that trusts the prefix and does a
blind `read_exact(len)` on the body hits EOF and **errors the whole
stream** — so a single partial tail throws away **every complete
record before it**, and a crash-truncated log fails to replay / upload
*entirely* instead of recovering the 99% that's intact.

- **Treat a mid-record EOF as a clean end-of-stream, not an error.**
  When the body read returns fewer bytes than the prefix promised at
  the *end* of the stream, stop and return "no more records"
  (`Ok(None)`) — you've recovered every complete record and dropped
  only the unfinished tail. Reserve a hard error for corruption in the
  *interior* (a torn record with valid records after it is real
  damage, not a clean truncation).
- **The last record is always suspect; design for it.** This isn't an
  exotic edge case — any writer that can crash produces it, so the
  reader's contract *includes* the partial tail. A length-prefixed or
  self-delimiting framing makes "is this record complete?" checkable
  before you consume it; a checksum per record turns "torn" into a
  positive test.
- **Test it with a deliberately truncated stream.** Two good records
  plus a third cut mid-body, asserting the reader yields the two and
  ends cleanly — the §18.5 RED you'd otherwise only hit in the field
  after a rig hard-powers-off.

The generalisable rule: **a reader of an append-only record stream
must survive a torn trailing record left by a writer killed
mid-append — a mid-record EOF at the end of the stream is a clean stop
that recovers every complete record, never a `read_exact` error that
discards the whole log; the last record is always suspect, so make
completeness checkable (length prefix / checksum) and test against a
truncated stream.** See anti-pattern #111.

---

<a id="validation-surfaces"></a>
## 17. Validation surfaces

> **When this fires** — a workflow has a canonical entry point (an onboarding dialog, a bootstrap script, a first-enroll flow).
>
> **Do** — fix the **canonical path**; a side-channel that works while the canonical path fails *is* the bug, not a workaround to ship.

When a workflow has a **canonical entry point** (an onboarding
dialog, a bootstrap script, a first-enroll flow), that entry
point **is** the validation surface. Side-channels that bypass it
(`scp + ssh ./push.sh`, ad-hoc REST calls, manual file writes)
are debugging tools, not workflows.

If something works through a side-channel but fails through the
canonical entry point, the canonical entry point is the bug. Fix
it there; don't ship "you have to push by SSH" as a workaround.

---

<a id="tests"></a>
## 18. Tests / verification

> **When this fires** — writing or fixing code with a testable contract, or about to claim 'done.'
>
> **Do** — **test-first** (RED → GREEN → REFACTOR); state what you ran and what it told you; a wrapper propagates the real exit code; settle a decision with recorded evidence.

State what you ran and what it told you. "Tests pass" without a
command and an exit code is not a verification — it's a wish.

If you couldn't run the test that would actually prove the change
(needs real hardware, a vehicle bus, a multi-camera rig, a GPU,
network access to a fleet endpoint), **say so explicitly**. Do
not claim success from unit tests alone for a change that touches
hardware behaviour.

For interactive surfaces: launch the surface and exercise the
path. If you can't, say so. Provide a three-tier selftest where it
makes sense — (1) imports + service constructors, (2) a fast
in-process integration check, (3) a frozen-bundle invocation that
catches build-spec issues source can't see. The frozen-bundle
tier has caught build-only bugs (missing wheels, mis-included
data files) that no source-tree test exercises.

<a id="smoke-probes"></a>
### 18.1 End-to-end smoke probes as runnable scripts

Some flows cross too many components to unit-test cleanly:
"operator picks a new target → transport rebuilds → subscribers
re-subscribe → supervisor transitions → first frame arrives."
Each step is testable; the *sequence* isn't, easily.

For these, ship a `scripts/_smoke_*.py` that traces the path
**against the real backend**, reporting observed state at each
step. Not a unit test — a probe a human runs after a refactor to
catch the "I broke step 4" class of regression:

```python
# scripts/_smoke_swap.py
log.info("─── building app_state ───")
state = build_app_state(config)
assert state.transport is not None
log.info("transport=%s", state.transport.kind)

log.info("─── swapping to %s ───", target_id)
swap_target(state, target_id)
assert state.transport.kind == "webrtc"
log.info("subscribers re-subscribed: %s", state.subscriptions)
...
```

Each `assert` localises the failure to a step; each `log.info`
shows the observed state so a failed assertion is debuggable from
the journal alone. The smoke is **runnable in isolation** —
operator runs it after a code change touching the swap path,
before re-launching the full UI.

**Probe a representative payload, not a tiny sentinel.** A probe
that pushes a 1 KB sentinel through an upload / transfer / encode
path passes green while the **real** multi-MB payload fails — the
failure mode (a timeout, a buffer cap, a fragmentation boundary, a
bandwidth limit, §9.9) only appears at real size. We shipped this:
a green "upload works" probe used a tiny blob while every real
9 MB segment 502'd. Make the probe exercise the **size and shape**
the production path actually carries (a big-blob case alongside the
small one), or it's testing a path the workload never takes.

<a id="migration-completeness"></a>
### 18.2 Migration completeness — prove the port carried everything

When you migrate a body of code — a framework swap (Qt → ImGui),
a language port, a service reimplementation, a rename sweep — "the
target runs" is **not** "the migration is done." A function can
silently fail to make the crossing, and you discover it months
later when something quietly doesn't work. A migration is done
only when **every source unit is provably either ported or
consciously dropped.**

**The drift trap (this is the one that bites).** The moment you
start adding *new* functions to the target after the migration,
the target and source diverge further, and you can no longer eyeball
"what was original vs. what I added." A dropped function becomes
invisible — it's lost in the gap between an old source and a
drifted target. So:

> **Run the completeness check against the SOURCE AT ITS
> PRE-MIGRATION STATE, not against today's target.** Every day of
> added functions makes a later audit harder, not easier. If you
> already migrated and added a lot since — do this now, it only
> gets worse.

**The method:**

1. **Reconstruct the source surface from git at the migration
   base.** You don't need a manifest saved in advance — git has
   it. Enumerate the source's public units (functions, classes,
   services, dialogs, endpoints, CLI commands, message handlers)
   as of the pre-migration commit:

   ```sh
   # source public surface at the migration baseline
   git -C <source> grep -hoE '^\s*(def|class|fn|pub fn) \w+' \
       <migration-base> -- <paths> | sort -u > /tmp/source_surface.txt
   # current target surface
   grep -rhoE '^\s*(def|class|fn|pub fn) \w+' <target>/ \
       | sort -u > /tmp/target_surface.txt
   # in source, absent from target → the audit worklist
   comm -23 /tmp/source_surface.txt /tmp/target_surface.txt
   ```

2. **Triage every line of that worklist** into exactly one of
   three buckets — none may be left blank:
   - **ported-renamed** — same behaviour, new name. Add it to the
     mapping table so the next audit doesn't re-flag it.
   - **consciously dropped** — record *why* (obsolete, replaced by
     a different mechanism, never used). A dropped unit with a
     reason is fine; a dropped unit nobody decided to drop is the
     bug.
   - **regression** — genuinely missing. Port it.

   (The comm-output is a *worklist to triage*, not a bug list —
   renames are expected false positives. The point is that nothing
   reaches "done" untriaged.)

3. **Capture the decisions in a mapping table** — the
   "what's reused / what's rewritten / what's dropped" table is
   the migration's contract. Each source unit → target unit, or
   → DROPPED (reason). This table is also what makes the *next*
   person's audit a diff instead of a re-derivation.

4. **A parity test makes it permanent.** A check that fails when a
   source symbol is neither in the target nor on the explicit
   dropped-list turns "did we port everything" from a one-time
   heroic audit into a button. New target functions don't trip it
   (additions are fine); only an *unaccounted source unit* does.

**Schemaless payloads migrate silently.** Renaming a key in an untyped
telemetry / heartbeat dict (`network_type` → `uplink_kind`) is a migration
too — but there's no compiler to flag the consumers you missed, so an
un-updated reader gets `""` (a falsy, no-error read) and quietly fails its
gate. Sweep **every** consumer across repos (grep the old key fleet-wide),
read with an old-name fallback through the transition, and treat the
payload as an append-only schema ([§4](engineering-rules.md#schemas)). See
anti-pattern #125.

The generalisable rule: **a migration's done-criterion is a
two-sided inventory, not a passing smoke test.** Behaviour tests
prove what you ported works; only a source-vs-target surface diff
proves you ported *all of it*. See anti-patterns #28, #125.

<a id="decision-record"></a>
### 18.3 Settle a decision with recorded evidence; re-open it only on new evidence

The "back-and-forth" — a choice made, reverted, re-made across
weeks — is rarely a hard problem. It's an **unrecorded** one: the
decision was never written down *with the evidence that settles
it*, so it isn't actually settled, and anyone can swing it back on
a hunch. The cure is decision hygiene, and it's the same three
moves whether the choice is CPU-vs-GPU
([§9.7](engineering-rules.md#cpu-gpu-decision), [§9.10](engineering-rules.md#validated-accelerator-default)),
a library, a default value, or an architecture.

**1. Decide once, by attributed evidence — not by feel.** What ends
an oscillation is a *measurement attributed to a cause*
([§9.7 r1](engineering-rules.md#cpu-gpu-decision)) or a validation *on the target*
([§9.10](engineering-rules.md#validated-accelerator-default)) — not "seems faster" /
"feels safer" / "I'd have done it differently." A decision taken on
feel isn't settled; it's the next person's coin-flip.

**2. Record the verdict, its evidence, its blocker, and the bar to
re-open.** One durable note where the thing lives — next to the
code, in the process-table policy ([§6.1](engineering-rules.md#service-inventory)), in
the migration map ([§18.2](#migration-completeness)): *what was
chosen, the numbers that chose it, why the alternative was shelved,
and the precise new evidence that would re-open it.* Then "should
we switch this?" is answered by **reading**, not by re-running the
loop from zero. An undocumented revert is how the loop restarts.

**3. The recorded default wins until new evidence overturns it.** A
change back across a settled decision must clear the *same bar that
set it* — a new attributed measurement that overturns the record —
and must update the record. "Let's just try the other way again"
with no new evidence is re-litigation: decline it and point at the
note. A *silent* revert (a fallback that quietly takes the
abandoned path) is worse — it's
[authoritative-vs-actual divergence](anti-patterns.md#recurring-shapes)
between the recorded decision and what's running, so make any
fallback a **loud, surfaced degraded mode**, never the silent
steady state ([§9.10](engineering-rules.md#validated-accelerator-default)).

The generalisable rule: **a contested or reverted decision is
settled by a recorded verdict — the evidence that chose it, the
blocker, and the bar to re-open — and that record is the default
until new evidence overturns it; oscillation is the signature of a
decision that was never written down, not of a hard problem.** This
is the sixth [recurring shape](anti-patterns.md#recurring-shapes),
the general form of [§9.7](engineering-rules.md#cpu-gpu-decision) /
[§9.10](engineering-rules.md#validated-accelerator-default). See anti-pattern #42.

<a id="propagate-exit-code"></a>
### 18.4 A wrapper's exit code must reflect the work — a script that always exits 0 is a false green

If an exit code isn't proof a change *works* ([§18](#tests) — "an
exit code is not a verification, it's a wish"), it's at least the
machine-readable verdict that **CI, a supervisor, and a scheduler all
gate on**. So a wrapper / launcher / build script must **propagate
the real exit status of the work it runs.** A script that ends on a
`echo "done"`, an unchecked pipeline, or a swallowed `|| true` returns
**0 regardless** — and every automated consumer reads that 0 as
success while the actual work failed. A green that lies is worse than
a red.

We shipped this twice. A build wrapper ran the real on-rig build, then
ended with a status line, so its exit code was **always 0** — a failed
build reported success and CI stayed green. And a scheduled bump job
ran its inner steps but returned the last `echo`'s status, so launchd
recorded `LastExitStatus = 0` even when pushes failed.

- **Capture and return the child's status.** `cmd || rc=1` across the
  steps and `exit $rc` (or `set -e` / `set -o pipefail` so a failure
  aborts loudly). A wrapper's last line is usually an `echo`, whose 0
  then masks everything above it.
- **A pipeline's status is the *last* command's** unless you set
  `pipefail` — `make 2>&1 | tee log` exits 0 even when `make` failed.
- **Verify the propagation, not just the happy path:** force the inner
  step to fail and confirm the wrapper (and its CI/supervisor) goes
  red. An untested error path is dead code ([§18.1](#smoke-probes)).
- **The complement: don't let a *benign* non-zero abort the work.**
  `set -e`/`pipefail` is right for the *essential* steps, but it also
  kills the script on a non-zero that **isn't a failure** — a glob that
  legitimately matches nothing (`ls -t build/*.whl` on a clean tree), a
  `grep` with no hits, an **optional** file that an older checkout
  predates, a best-effort probe. We shipped both ends: a wheel-freshness
  `ls -t` glob with no match **aborted the whole build**, and an
  onboard's `install <optional>.py` hard-failed (`cannot stat …`) on a
  bundle that legitimately lacked the newer sibling — webrtcd never came
  up. Guard the **genuinely-optional / may-be-empty** step (`… || true`,
  an `[ -f ]` / `[ -n "$(…)" ]` existence check, consume-when-present)
  so it **degrades and logs** instead of aborting — *without* blanket
  `|| true` on the real work (that's the §18.4 lie). The discriminator
  is "is this step the work, or auxiliary to it?": the work fails loud;
  an absent optional or an empty glob degrades.

The generalisable rule: **a wrapper / launcher / CI script exits with
the real status of the work it ran — never a blanket 0 from a trailing
`echo`, an unchecked pipe, or a swallowed error; a script that's green
no matter what trains every automated consumer to trust a lie. The
complement under `set -e`/`pipefail`: guard a genuinely-optional or
may-be-empty step (an empty glob, an absent optional file, a best-effort
probe) so a *benign* non-zero degrades instead of aborting the build —
the essential work still fails loud.** See anti-patterns #77, #117.

<a id="test-first"></a>
### 18.5 Write the test first, and watch it fail — RED → GREEN → REFACTOR

For a change with a **testable contract** (most logic — a parser, a
transform, a state machine, a bug fix), write the test *before* the
code, in three moves:

1. **RED — write the test and run it; *see it fail* for the right
   reason.** A test you never watched fail proves nothing: it may
   assert the wrong thing, hit the wrong code, or be vacuously true. A
   bug fix starts with a test that **reproduces the bug** (red now);
   that red is your proof the test actually exercises the path
   ([§15 diagnostic-first](#diagnostics) — reproduce before you fix).
2. **GREEN — write the *minimum* code to pass.** No speculative
   structure, no second case that isn't tested
   ([Implement: simplicity first](CLAUDE.md)). The test, not your
   imagination, defines "done."
3. **REFACTOR — clean up under a green bar.** Now simplify; the test
   holds the behaviour still while you improve the shape.

This is the engineering form of *evidence over claims*: the change is
proven by a test that went **red then green for an attributed reason**,
not by "looks right." It's also the antidote to the dead-recovery trap
([anti-pattern #23](anti-patterns.md) — a path never seen to fire) at
the test layer: a test never seen red is dead code that always passes.

- **A test written *after* the code, never seen red, is suspect** — it
  tends to encode whatever the code already does, bugs included. If you
  must add a test to existing code, break the code on purpose once and
  confirm the test fails, then restore.
- **For paths you can't unit-test (hardware, a fleet endpoint, a GPU,
  a real-time loop)** the discipline still holds in analog form: state
  the **observable success signal and the failing baseline first**
  (the rate, the log line, the wire value you expect — and what it
  reads *now*, broken), then change code until the signal flips. Say so
  when the proof is a live observation, not a unit test
  ([§18](#tests) — state what you ran).

The generalisable rule: **for anything with a testable contract, write
the test first and watch it fail for the right reason before you make
it pass, then refactor under green — a test never seen red proves
nothing, and "minimum code to green" keeps you from building what no
test asked for.** See anti-pattern #78.

<a id="regression-guard"></a>
### 18.5.1 HARD CONSTRAINT — every bug fix ships a PERMANENT guard; you may never trade a new fix for an old one

The recurring failure this kills: **you fix bug B and an already-fixed
bug A silently comes back** — the "whack-a-mole / back-and-forth" where the
same defect keeps returning because nothing *held it down*. It happens for one
reason: **fix A left no executable guard**, so a later, unrelated change (a
refactor, an auto-fixer, a second fix with an opposing requirement) quietly
reverted it and nothing went red. The fix lived only in a commit message and a
memory — and memory is not enforcement.

Three rules, all HARD (a change that breaks any one does not land):

1. **A bug fix is not done until a permanent guard would go RED if the bug
   returns.** The regression test from [§18.5](#test-first) is not a one-time
   proof you delete — it **stays in the suite forever** as the fix's executable
   memory. No guard, no fix: "I fixed it" without a test that reds on
   reintroduction is an *unenforced* fix, i.e. a future regression waiting to
   happen. For paths you can't unit-test, the guard is the strongest available
   equivalent — a runtime `assert`, a CI/lint check, a startup self-test, a
   `test_no_stale_*`-style scanner — never "we'll remember."
2. **When a new change would reintroduce an old bug, the old guard reds — STOP
   and reconcile BOTH requirements; never delete/loosen the guard to get
   green.** That red bar is the system working: it caught you trading fix A for
   fix B. The two requirements are a *constraint pair* — satisfy both, or if
   they are genuinely irreconcilable, that is a design decision to escalate with
   the trade-off named, not a test to silence. Deleting or `xfail`-ing a
   regression guard to land a fix is the single most direct cause of the
   ping-pong; it is forbidden.
3. **Deliberate-looking-wrong code carries a marker naming WHY + its guard, so a
   refactor or auto-fixer can't "clean it up" and revert the fix.** Fixes often
   look like mistakes: a re-exported symbol nothing local uses, a magic
   constant, a specific list ORDER, a redundant-seeming check, a workaround for
   someone else's bug. A tool (`ruff --fix`, "remove unused", a formatter) or a
   well-meaning cleanup will *undo* it — reintroducing the bug — unless the code
   says, in place, "this is load-bearing." Mark it: a `# noqa: <rule>` /
   `// keep:` with a one-line reason AND a pointer to the guard test or the
   failure mode. If it's not worth a marker, it's not worth reverting-by-accident
   either.

The generalisable rule: **a fix isn't finished until a permanent guard reds on
its reintroduction; when that guard fires against a later change you reconcile
both requirements instead of silencing it; and any fix that looks like a mistake
is marked in place with its reason + guard so no tool or refactor can quietly
revert it.** A bug that keeps coming back is not a hard bug — it is an
unguarded one. See anti-pattern #126.

<a id="mock-discipline"></a>
### 18.6 Mock discipline — test the real thing; a mock you must use is complete, understood, and never the assertion target

A mock is a necessary evil, not a convenience — every mock is a place
where the test stops testing reality. Four disciplines keep mocked
tests honest:

- **Prefer the real thing; mock only the genuinely slow / external
  boundary** (the network call, the hardware device, the clock) — not
  the high-level method that happens to be convenient. And before you
  mock anything, **understand its side effects**: mocking a call whose
  side effect the code under test depends on makes the test pass for
  the wrong reason or fail mysteriously.
- **Never assert on the mock — assert on the behaviour.** A test that
  checks "the mock was called" / "the mock is present" verifies the
  *mock*, not the component. It stays green when the real integration
  is broken — the mock-flavoured form of the never-seen-red trap
  ([§18.5](#test-first), anti-pattern #78).
- **A mock response mirrors the real structure completely.** A partial
  mock ("just the fields I think it needs") hides the structural
  assumption; the test passes while real integration fails on the
  missing field. Build the mock from the documented / captured real
  response, all fields.
- **No test-only methods on production classes.** A `destroy()` /
  `reset()` that only tests call contaminates the production surface
  and is a loaded gun at runtime. Cleanup helpers live in test
  utilities, not on the class.

The generalisable rule: **mock only the slow or external boundary and
only with its side effects understood; assert on real behaviour, never
on the mock; mirror the full real structure; and keep test-only
affordances out of production classes — a test that asserts on its own
mock is green regardless of whether the system works.** See
anti-pattern #79. (From the testing anti-patterns in
[obra/superpowers](https://github.com/obra/superpowers), adapted.)

<a id="finish-the-operation"></a>
### 18.7 An automating tool is "done" at the state the next consumer reads — committed, not written

A bootstrap, installer, scaffolder, or publisher hands its output to a
*next* tool — a sync tool advances a **committed** submodule pin, CI
builds from **HEAD**, a supervisor runs what's **installed**, a registry
serves what was **pushed**. Its job is **not** done when the files are
written; it's done when they're in the **state that next consumer
reads**. Stop one step short — vendor but don't commit, build but don't
push, generate but don't register — and you leave a **half-done
artifact**: invisible to the downstream tool, and dependent on a "now
run these steps by hand" note that gets forgotten.

We shipped this: a repo-bootstrap script vendored a submodule and wrote
the symlinks but **stopped before committing**, printing a manual
`git add && commit`. The repo sat staged-but-uncommitted — so the sync
tool that manages the pin couldn't see it, and it stayed half-set-up
until someone noticed. The fix was to make the tool **finish**: commit
the artifact itself — kin to a publish step that builds then also
commits + pushes + updates the registry, done at *rollout*, not at
*compile* ([§28.1](engineering-rules.md#roll-out-shared-fix)).

- **Carry the operation through to the consumer's contract.** If the
  next tool reads a committed pin, commit it; a pushed tag, push it; a
  registered entry, register it. "It ran" / "the files are there" is
  [§18.2](#migration-completeness)'s *it-runs ≠ it's-done* in another
  guise.
- **The finishing git step is path-limited to the files you own.** Stage
  an **explicit allowlist** of the artifacts (`git add -- a b c`) and
  commit *those paths* — **never `git add -A`**, which sweeps a
  co-located human's WIP into your machine commit
  ([§20.1](#merge-to-main), [§19.1](#allowlist-not-glob)).
- **Make it idempotent — and key the "skip, already done" check on the
  consumer-visible end-state, not a local marker.** A re-run with nothing
  new to do does nothing. But if it decides "already done, skip" by
  comparing a *local* signal (the committed pointer, a written file) to
  the target instead of the *published* one (`local == remote`, the
  applied value, the served artifact), then a prior run that finished
  **locally** but failed to **publish** — a dropped push on a flaky link
  — is declared complete: `local == target`, "nothing to do," so the
  unpushed tail is **stranded across every re-run**. Gate "done" on the
  published state so the re-run re-publishes the gap; and when a push
  fails mid-fan-out, re-push that stranded commit *directly* (the
  skip-on-local-state tool never will). This is *committed ≠ pushed*
  ([§12.1](#build-version-stamp)) inside the sync tool itself.

The generalisable rule: **an automating tool's work is done at the state
its downstream consumer reads — committed / pushed / registered — not
when the files are written; finish the operation through that handoff
(the finishing commit path-limited to the artifacts it owns, never
`add -A`), and make it idempotent with the "skip, already done" check
keyed on that same consumer-visible end-state — so a half-done
intermediate is never completed by hand, and a prior run that finished
locally but failed to publish is re-published, not declared done.** See
anti-patterns #87, #88.

<a id="rewrite-parity"></a>
### 18.8 Prove a rewritten transform matches the original — dual-run on real data

[§18.2](#migration-completeness) proves the port carried every *unit*
across (surface coverage). It does **not** prove each ported unit
produces the **same answer**. When the rewrite's job is to *reproduce*
an existing transform — a wire bridge, a decoder/codec, a filter or
state estimator, a model path, a unit conversion — "it compiles", "it
runs", and even "its unit tests pass" do not establish that it emits the
**legacy's** output. A subtly wrong constant, a dropped term, a rounding
or endianness difference, an off-by-one in a window produces a clean run
with quietly different numbers, and the divergence surfaces downstream as
a controls/perception bug nobody traces back to the port.

- **Dual-run on real recorded inputs.** Feed the **same** real captured
  data ([§16.9](#one-record-format)'s log of record is exactly this
  corpus) through both the legacy and the rewrite, and assert the outputs
  match — **bit-identical** where the path is deterministic, **within a
  stated tolerance** for float / seeded / reordered results (state the
  tolerance and *why* it's acceptable). Synthetic unit inputs miss the
  regimes real data exercises.
- **Run it before cutover, keep it as a regression guard.** Parity is a
  *gate* on trusting the new path, not a one-time demo — wire it into the
  build/CI so a later edit that breaks equivalence fails loudly. A passing
  parity run on real data is what lets the [decision record](#decision-record)
  say the rewrite is validated; without it the new path is UNVALIDATED and
  ships only gated-inert ([§20.1](#merge-to-main)).
- **A mismatch indicts the rewrite, not the legacy.** The legacy is the
  reference by definition; when they differ, the new code is wrong until
  proven otherwise — diff the intermediate stages
  ([§15.8](#hypothesis-discipline)) to localise where the two paths
  first diverge.

The generalisable rule: **when a rewrite's job is to reproduce an existing
transform, prove behavioural parity by running the legacy and the new code
on the *same real recorded data* and asserting equal output (bit-identical
or within a stated tolerance) — "it runs" and "unit tests pass" don't prove
it produces the legacy's answer; gate cutover on the parity run and keep it
as a regression guard.** See anti-pattern #109.

<a id="inject-the-fault"></a>
### 18.9 Validate a recovery path by injecting the fault — it never runs in normal operation

A recovery / fallback / safe-state branch executes only under a failure
that, by definition, doesn't happen in dev: a dropped link, a diverged or
"kidnapped" estimate, a torn file, an exhausted slot, a reaped peer, a
stale override. So it is the **least-exercised code you most need to
work**, and both the recovery *and its trigger* rot silently. The trap
that hides it: the recovery body looks correct in review, but the
**detector meant to fire it is broken** — a lost-state trigger that never
trips, a threshold never crossed, an event the upstream stopped sending
(anti-pattern #23) — so the branch is dead code and nobody notices until
the failure finally arrives in the field.

The discipline: **deliberately inject the fault, assert the system both
*detects* and *recovers*, and keep the injection as a regression test.**

- **Inject the real failure, not a mock of it.** Kill the link, teleport /
  "kidnap" the estimate, truncate the file mid-record, exhaust the bounded
  store, `SIGKILL` the writer, hand over a stale override. A hook that
  produces the *actual condition* exercises the **detector** too, not just
  the recovery body.
- **Assert detection AND recovery — two checks.** The trigger fired (a
  counter at the recovery entry incremented, #23) *and* the system
  returned to a good state (re-acquired lock, resumed fresh data, dropped
  to the safe default). A correct recovery body behind a dead trigger
  still fails.
- **Keep it as a runnable injection / regression test.** The branch will
  rot again on the next refactor; a kept fault-injection (a unit test, a
  sim "kidnap" mode, a chaos toggle) is what keeps it alive. "We have a
  recovery path" is a claim; "the injected fault recovers, in CI" is the
  proof.

This is the active form of [§18.5](#test-first) for the branch that never
runs on its own, and it validates the recovery rules elsewhere in the book:
safe-stop on a stale command ([§3.7](engineering-rules.md#command-link-safe-stop)),
fail-safe on a bad input ([§3.5](engineering-rules.md#validate-safety-input)),
torn-record recovery ([§16.10](#torn-record-recovery)), the singleton
restart race ([§6.8](engineering-rules.md#singleton-process)), bounded
reconnect ([§6.4](engineering-rules.md#bounded-reconnect)).

The generalisable rule: **a recovery / fallback / safe-state path never
runs in normal operation, so it and its trigger rot silently — validate it
by injecting the fault it recovers from (kill the link, kidnap the
estimate, truncate the file, exhaust the slot), asserting the system both
*detects* and *recovers*, and keep the injection as a regression test;
"we have a recovery path" ≠ "the injected fault recovers."** See
anti-pattern #23.

---

<a id="reversibility"></a>
## 19. Reversibility

> **When this fires** — a destructive op, a secret, a push, or a delete / cleanup that selects targets by a glob.
>
> **Do** — **ask first** for that specific destructive action; delete an explicit allowlist, never a wildcard; enforce a hard cap with a deterministic, self-contained key.

No `git push --force`, no `git reset --hard`, no `rm -rf` against
shared paths, no schema migration without a rollback plan —
without the user explicitly OK'ing **that specific destructive
action**, not the surrounding task.

Secrets (`*.gpg`, `*_private.*`, `.lfsconfig`, signed release
bundles, fleet tokens, signing keys) are not read, not staged, not
committed. If you stumble onto one, say so — don't keep going as
if you didn't see it.

Persistent config writes use **tmp + rename** for atomicity. A
half-written config file on the read side is a configuration
silently reverting to defaults; the right shape is:

```python
tmp = path.with_suffix(path.suffix + ".tmp")
tmp.write_text(json.dumps(data, indent=2))
tmp.replace(path)           # atomic on POSIX
```

<a id="allowlist-not-glob"></a>
### 19.1 A destructive cleanup names what it deletes — an allowlist, not a wildcard

A "clean up the X files" routine that selects its targets with a
**broad pattern** — `rm *.json`, `DELETE … WHERE name LIKE …`,
"remove everything in this directory" — deletes every **bystander
that merely matches** it, including co-located files that aren't in
scope. A directory (or table, or prefix) is a *shared* namespace; the
wildcard can't tell which entries the operation actually owns.

We shipped this: a calibration **wipe** globbed `*.json` in a
directory that also held two non-calibration files (a device-config
and a survey artifact) and deleted them too. The wipe "succeeded" and
silently took out config that nothing else restores.

- **Select by the known set, not a pattern.** Delete the *enumerated*
  artifacts the operation owns (`{a,b,c}.json`), not everything that
  matches a glob. If the set is dynamic, derive it from an
  authoritative manifest — never from "whatever's in the folder."
- **Assume a bystander is present.** Co-locating unrelated files in
  one directory is normal; a destructive sweep over a shared path is a
  §19 action — scope it so an innocent neighbour can't be caught.

The generalisable rule: **a destructive cleanup deletes an explicit
allowlist of the artifacts it owns, never a broad glob over a shared
directory — a wildcard sweeps in co-located bystanders that happen to
match, and a delete doesn't come back.** See anti-pattern #72.

<a id="hard-cap-deterministic-eviction"></a>
### 19.2 A hard-cap eviction is deterministic and self-contained — don't gate a must-hold limit on a best-effort signal

A bounded store (a disk of recordings, a ring of logs, a cache) has
two very different jobs, and conflating them breaks the important one:

- a **soft / opportunistic** sweep that *optimises* — free space
  early, prefer dropping already-synced data — where being selective
  is fine because being wrong is harmless; and
- a **hard cap** — the invariant you *must* hold ("never exceed 90 %
  disk") — where enforcement has to be **deterministic and depend
  only on inputs you control**.

The trap is gating the hard cap on a **best-effort / derived signal**.
We shipped this: a disk-cap deleter preferred evicting *uploaded*
segments first — so the 90 % cap depended on an upload-detection xattr,
and that signal was flaky (a stale-`None` cache). When it lied, the
cap couldn't be enforced: the store could march past its limit while
the deleter "protected" data it wrongly believed un-synced.

- **Enforce the hard cap on a self-contained key.** Evict in a
  deterministic order — oldest-first (FIFO) for a rolling buffer —
  using only reliable local facts (creation time, an operator lock),
  never a signal that can be wrong. At the cap, the oldest goes,
  period.
- **Keep the selective / optimising logic in the soft tier.** "Spare
  un-synced data" belongs in the opportunistic sweep, where a wrong
  signal just frees a little less — not in the must-hold cap, where a
  wrong signal means the invariant silently fails.

The generalisable rule: **a hard invariant (a cap, quota, safety
limit) is enforced by a deterministic rule over inputs you control —
never gated on a best-effort / derived signal that can be wrong;
reserve selective, signal-dependent logic for the soft / opportunistic
tier where being wrong is harmless.** See anti-pattern #74.

---

<a id="repo-hygiene"></a>
## 20. Repo hygiene

> **When this fires** — pushing, merging to main, or doing concurrent / risky work on a shared, live checkout.
>
> **Do** — right remote and branch; fast-forward only your **committed + validated** branch (never `git add -A`); isolate parallel work in its own **git worktree**.

- Run `git remote -v` before pushing into a project you haven't
  pushed to before. Push targets are not interchangeable across
  the company.
- Run `git symbolic-ref refs/remotes/origin/HEAD --short` (or
  `git remote show origin | grep 'HEAD branch'`) when you're not
  sure whether the default branch is `main` or `master`. Don't
  pattern-match.
- Commit messages describe *why*, not *what* — `git diff` already
  shows what.
- **Git over SSH is already configured for commit and push** — plain
  `git commit` / `git push` / `git push origin <tag>` work without a
  prompt, and **pushing an annotated tag over SSH *is* publishing the
  version**. The GitHub CLI (`gh`) uses a *separate* token auth
  (`gh auth login` / `GH_TOKEN`) that may not be set up; an
  unauthenticated `gh release` failing is a *missing token*, **not a
  broken push**. So when `gh` errors with a login prompt, **don't
  retry-loop it** — the SSH `git push` already published; `gh release`
  only adds the web release page, so skip it cleanly and report it as
  optional rather than retrying. (Same retry-loop trap as a throttled
  cloud edge, [§23.3](#edge-new-conn-throttle): a one-shot failure
  against the *wrong* mechanism reads as "broken" when the real path
  already succeeded.)

<a id="merge-to-main"></a>
### 20.1 Feature branches and merging to main on a shared checkout

These checkouts are **shared and live** — at any moment a working
tree holds a *mix*: your committed feature branch, collaborators'
branches, and large uncommitted WIP that may be theirs or just
unfinished. Develop on a branch (`feat/…`, `fix/…`), keep `main`
always releasable, and when you merge take **only your own
validated, committed work** — nothing else rides along.

**Merge only what's committed *and* validated.** Validated =
committed by you *and* proven (live-tested, or recorded as
validated). Uncommitted WIP is neither — surface it to its owner,
don't decide it's ready. Never `git add -A` / commit the working
tree "to merge everything": that sweeps collaborators' and
unfinished work onto `main`.

**Fast-forward off-checkout; don't disturb the working tree.** Push
your committed branch straight to main without checking main out:

```sh
git push origin <feature>:main      # FF only — no --force
git branch -f main <sha>            # sync the local pointer; stay on the feature branch
git status                          # same WIP as before — you disturbed nothing
```

No `--force`: if `main` diverged the push is *rejected* rather than
clobbering a concurrent push — the safe default (§19). Leave
collaborator branches and others' uncommitted files untouched.

**A build reads the working *tree*, not git — so commit before a
release build.** Build/deploy scripts that compile the current
working tree (not a clean checkout of a committed ref) bake in
whatever is lying around — your uncommitted edits *and* a
collaborator's WIP — so you ship code that exists on no commit. The
`-dirty` build stamp ([§12.1](#build-version-stamp)) is the
tripwire; the fix is to **commit (and clean) before a release
build**, ideally building from a fresh checkout of the exact ref
you intend to ship.

**Landing *unvalidated* work safely — gate it inert, don't hold it
back.** "Merge only validated work" doesn't mean a risky or
not-yet-proven capability can never reach `main`; it means it must
be **inert by construction** until its validation lands. A
not-yet-validated path may merge **only if it can't act** — dark
behind a gate that defaults off and only arms when its proof is
present (a control feature whose engage-gate derives from a
calibration that isn't checked in yet, [§3.5](engineering-rules.md#validate-safety-input)
/ [§9.10](engineering-rules.md#validated-accelerator-default)) — committed, and the
commit **labelled UNVALIDATED**. The *gate* must be validated even
when the feature isn't; the safe default (§9.10) is the gated-off
state. What you must never merge is an unvalidated path that can
actually **act** — that's not a dark launch, it's shipping unproven
behaviour.

The generalisable rule: **on a shared checkout, `main` gets only
your committed, validated work, fast-forwarded without disturbing
the working tree; never `add -A` WIP onto main, and never cut a
release build from a dirty tree — a build compiles the working
tree, not the branch you think you're on. An unvalidated capability
may land only gated inert until its proof arrives (the gate
validated, the feature dark, the commit labelled).** See
anti-pattern #59.

<a id="worktree-isolation"></a>
### 20.2 Isolate concurrent, parallel, or risky work in a git worktree

[§20.1](#merge-to-main) is the *reactive* discipline for a shared,
live checkout — don't sweep someone else's WIP onto main. A
**git worktree** is the *proactive* cure: it gives each branch / task
/ experiment its **own working directory off the same repo** (one
shared `.git`, separate index and files), so concurrent work simply
**can't collide** — there is no shared tree to stash around, fight, or
`add -A` by accident.

```sh
git worktree add ../<repo>-<branch> <branch>   # new tree for a branch
git worktree add -b feat/x ../<repo>-x         # ... creating the branch
git worktree list                              # what's checked out where
git worktree remove ../<repo>-x                # dispose when done
```

Reach for one when:

- **Another session / agent is editing the same repo.** Work in your
  own worktree and your `git status` shows *only your* changes — the
  "whose WIP is this?" hazard ([§20.1](#merge-to-main), anti-pattern
  #59) disappears by construction, and an automated harness can hand
  each task its own worktree (the same `isolation: worktree` idea).
- **A risky or large refactor** you want to keep apart from a clean
  tree, and **disposable** — `git worktree remove` and it's gone, no
  reflog archaeology.
- **A hotfix while a long task is mid-flight** — branch the fix in a
  fresh worktree instead of stashing a half-done tree.

Caveats: a branch can be checked out in only one worktree at a time;
**submodules need re-init in the new tree** (`git submodule update
--init`, which matters for the `.unocoding` pin); clean up with
`git worktree remove` / `prune` so stale trees don't accumulate.

The generalisable rule: **for concurrent, parallel, or risky-isolated
work — especially on a shared / live checkout — give each branch or
task its own `git worktree` so working trees never collide; it's the
structural fix for the shared-tree "whose WIP is this?" hazard
([§20.1](#merge-to-main)) and a disposable sandbox for experiments.**
See anti-pattern #59.

<a id="finish-the-branch"></a>
### 20.3 Finish the branch — merge it, park it with a recorded blocker, or delete it; a long-lived unmerged branch is unrecorded divergence

[§20.1](#merge-to-main) is *how* to merge safely; this is *that you
must finish*. A feature branch is **a loan, like a hot-edit on a box
([§13.3](#capture-live-edits))**: it exists to develop one change and
then **resolve** — and every day it stays unmerged, the system gets
harder to track. The fix everyone assumes is "in" answers differently
on `main`; the *checkout* parked on the branch makes every tool and
session that visits the repo see the branch's reality instead of the
default's; and across a fleet of repos, five different parked branches
mean **nobody can say what the system is** without an archaeology dig.

**"Commit" means *finish the work*, not `git commit`.** When the
instruction is "commit" (or "always commit"), the default is the whole
sequence to a clean terminal state — **commit → fast-forward to the
default branch ([§20.1](#merge-to-main)) → push → delete the
now-merged feature branch → return the checkout to the default
branch** — not a dangling commit left on a fork. Stopping at `git
commit` manufactures exactly the unmerged branch this section exists to
prevent. The other two terminal states below (parked-with-a-record,
deleted) are the deliberate exceptions, chosen *out loud* — silence
defaults to merged-and-cleaned. The same expands the verbs around it:
"push" carries the merge; "merge" carries the branch delete and the
checkout return. Finish the operation ([§18.7](#finish-the-operation)),
don't stop at the first verb in the phrase.

Every branch is in exactly one of three legitimate states — anything
else is drift:

1. **Merged.** Validated → fast-forward to the default branch
   ([§20.1](#merge-to-main)) → **branch deleted, checkout returned to
   the default branch**. The default branch is where tools, bumps, and
   the next session look; a finished branch that still holds the
   checkout is a trap (it's how a fleet pin "regressed" when a parked
   checkout switched under the tools). Delete **only your own** merged
   branch — on a shared checkout, a collaborator's merged-looking branch
   may be a handle they're still using ([§20.1](#merge-to-main)); leave
   it.
2. **Parked, recorded.** Genuinely blocked (awaiting on-rig
   validation, an external dependency)? Then it's a **recorded
   decision** ([§18.3](#decision-record)): pushed to the remote, the
   blocker and the merge-bar written down (commit message, the branch
   description, or the project notes). Prefer the
   [§20.1 gated-inert merge](#merge-to-main) where possible — the code
   lands dark on the default branch instead of rotting on a fork.
3. **Deleted.** Abandoned or superseded — delete it; an experiment's
   worktree goes with it ([§20.2](#worktree-isolation)).

**Track it like running-vs-declared ([§6.1](engineering-rules.md#service-inventory)):**
the unmerged set is enumerable — `git branch -vv --no-merged
<default>` per repo — so "what's unfinished?" is a **command, not a
memory**. An unmerged branch with no recorded blocker, or a checkout
sitting on a branch whose work already merged, is the finding.

The generalisable rule: **"commit" means *finish* — commit → FF to the
default branch (§20.1) → push → delete your merged branch → checkout
back to default — not a bare `git commit` on a fork. A branch is
finished promptly — merged (§20.1), parked with a pushed, recorded
blocker (§18.3), or deleted; a long-lived unmerged branch with no
recorded reason is silent divergence (authoritative-vs-actual at the
branch level), and the unfinished set must stay enumerable by one
command.** See anti-pattern #83.

<a id="production-branch-known-safe"></a>
### 20.4 The production branch stays on the last known-safe behavior — exclude a known-unsafe regression commit regardless of recency

> **When this fires** — cutting a production / release branch, or
> merging toward `main` on a fleet that runs from the default branch.
>
> **Do** — base production on the last commit known to be safe; if a
> regression has landed since, exclude that commit even if it's the
> newest — recency does not override known-unsafe.

The production branch's job is to run code the fleet has *validated*,
not the newest code on `main`. A known-unsafe regression that landed
after the last known-safe commit is a **disqualifier** for production
regardless of how recent it is — "but it's the latest" is not a safety
argument. The last known-safe commit is the floor; anything above it
that is known-unsafe is excluded, and the branch stays at the floor
until the regression is fixed (a new commit that restores safe
behavior) or explicitly reverted.

This is the **branch-level** twin of [§13.3](#capture-live-edits) (a
hot-edit is a loan captured back to the repo): the production branch is
a **loan from the repo's validated history**, not a pointer at
`HEAD`. Treating `main` as automatically production-safe is the same
shape as treating a dirty-tree build as a release
([§12.1](#build-version-stamp) dirty stamp) — both ship code the fleet never
validated. When a regression is identified:

1. **Identify the last known-safe commit** — the most recent commit
   *validated on the target*, not the most recent commit on `main`.
2. **Exclude the known-unsafe commit** — do not merge or deploy it;
   if it's already on `main` and `main` feeds production, the fix is
   a revert (a new commit), not `git reset` (rewrites history and
   strands downstream consumers, [§20.1](#merge-to-main) no-force).
3. **Record the exclusion** ([§18.3](#decision-record)) — which
   commit, why it's unsafe, what the blocker is — so the floor is
   known and the exclusion is auditable, not a folklore "don't ship
   the latest."

The generalisable rule: **the production branch is the last
known-safe commit, not the newest; a known-unsafe regression that
landed after it is excluded regardless of recency — recency is not a
safety argument, the validated commit is the floor, and the exclusion
is recorded (§18.3) so the floor is auditable.** See anti-pattern #83
(unmerged divergence) for the unmerged-branch twin and anti-pattern
#42 (validated-default) for the §9.10 accelerated-path analogue.

---

<a id="python-env"></a>
## 21. Python environment management

> **When this fires** — adding or changing a Python dependency, or
> provisioning a box to build/run a Unomove repo.
>
> **Do** — `unouv add` / `unouv ensure` and commit `uv.lock` — **never** a
> bare `pip install`. On a fresh box, `unouv doctor` then `unouv capnp`.

**`uv` is the default Python package + environment manager across
every Unomove project — and on company machines you drive it through
`unouv`, not bare `uv`.** Not `pip` into a hand-rolled venv, not
conda, not poetry, not pipenv. One manager everywhere is the only
way an environment tracks across the machines a project touches —
the dev laptop, the embedded target, the CI runner, the next
engineer's laptop.

`unouv` (`unolib/bin/unouv`, symlinked onto PATH) is a thin wrapper
that IS that one manager: it bakes in the lessons that bite every env
behind the GFW so `uv` "just works" — a **pinned** interpreter (never
the box's random `python3`), the aliyun / gh-proxy mirrors (PyPI, the
python-build-standalone CPython download, HuggingFace, torch's
CUDA-matched wheels), parallel downloads, and macOS TCC-stable code
signing of the venv Python. Under the hood it's still `uv`, so the
contract below (`pyproject.toml` + committed `uv.lock` + `uv sync`) is
unchanged — `unouv sync|add|run|python …` pass straight through with
the fleet env set. Reach for bare `uv` only when you've deliberately
opted out of the GFW-safe defaults (say why in `CLAUDE.local.md`).

**Native build tools count too — `capnp` is "always there" via
`unouv`.** The Rust schema crates (`unomsg`, `slam-msg`, …) shell out
to the Cap'n Proto **compiler** at build time (`capnpc::CompilerCommand`),
so a box missing `capnp` fails `cargo build` the same way a missing
interpreter fails a Python job — an un-provisioned env, not a code bug.
`unouv capnp` ensures it (builds it GFW-safe from the capnproto.org
release into `~/.local` when absent; idempotent), and `unouv doctor`
reports its presence. Run it once per box (or from a repo bootstrap);
don't hand-hunt a missing `capnp`.

The contract:

- **`pyproject.toml` declares dependencies. `uv.lock` pins the
  exact resolved graph, and `uv.lock` is committed.** The lock is
  the source of truth for "what is actually installed" — not the
  loosely-versioned `pyproject.toml` ranges.
- **`uv sync` recreates the locked environment byte-for-byte on
  any machine.** This is the cross-machine reproducibility
  guarantee. A teammate, CI, or the embedded target runs `uv sync`
  and gets the identical dependency graph the lock pins.
- **`.python-version` pins the interpreter.** `uv` reads it and
  provisions the right Python; nobody guesses whether it's 3.11
  or 3.12.

Canonical workflow:

```sh
uv venv -p 3.12 .venv            # provision the interpreter + venv
uv sync                          # install the EXACT locked graph
uv pip install -e '.[extra]'     # editable install with an extra

uv add <pkg>                     # add a dep: updates pyproject AND
                                 # the lock atomically
uv lock --upgrade                # deliberately re-resolve + re-pin
```

The rule that keeps the lock honest:

> **Adding a dependency is `uv add`, never a bare `pip install`
> into the venv.** A `pip install` mutates the live environment
> without touching `uv.lock`. The package works for you, the next
> `uv sync` (teammate / CI / target) doesn't have it, and you've
> shipped the "works on my machine" bug. Mutate the lock, commit
> it, push it — then the environment is portable. (See
> anti-pattern #22.)

Don't mix managers in one repo. A repo with a `uv.lock` is a `uv`
repo; ad-hoc `pip`, conda environment files, or a second lock
format desync the moment someone uses the other tool. If a
project genuinely can't use `uv` (a vendored build that hard-
requires conda, say), that's a deviation to **state explicitly in
the project's `CLAUDE.local.md`** and justify — not a silent
fork of the convention.

For embedded / cross-arch targets, `uv sync` resolves the same
graph against the target's platform; per-arch wheels (bundled or
from an index) are selected by `uv` the same way on every host.
The lock is the contract; the wheels are an implementation detail
`uv` handles.

---

<a id="new-service-checklist"></a>
## 22. When you add a new daemon, service, or topic

> **When this fires** — adding a new daemon, service, or topic.
>
> **Do** — give it a name, a consumer, a run-gate, and a written rationale — all four, or it isn't ready.

You owe the project four things:

1. A name in the uno-namespace, unambiguous, SI-unit-aware where
   it carries a quantity.
2. A consumer. If nothing reads the topic, don't ship it.
3. A gate predicate (for daemons) or a ring-size hint (for topics
   above the default 64 KiB).
4. A short inline rationale if any of the above is non-obvious.

If you can't supply all four, the addition isn't ready.

---

<a id="network-topology"></a>
## 23. Network topology

> **When this fires** — a network edge / tunnel / firewall, a web app behind a reverse-proxy subpath, or reaching a fleet host by IP.
>
> **Do** — the cloud edge is plumbing (reverse tunnel for off-LAN); relative URLs / one base path through the proxy; ask the registry's heartbeat-reported `lan_ip`, never a hardcoded IP or a subnet scan.

Cloud edge nodes (a rented host, a static-IP relay) are **network
plumbing**. They route. They do not host primary application
state, they do not run the operator-facing GUI, and they are not
the off-LAN entry point for a developer's laptop. Off-LAN access
to a host on the LAN is a reverse tunnel from a cloud edge port to
a LAN-side port, not a cloud nginx vhost.

Security-group / firewall config on cloud edge nodes is narrow on
purpose. Don't widen it for a one-off; extend the existing tunnel
service.

<a id="subnet-as-auth"></a>
### 23.1 "The subnet is the auth boundary" only holds if the edge provably can't reach in

A LAN-only internal service (a package registry, an admin port, a
metrics scrape, a debug endpoint) may legitimately run **without
its own auth** — *when the trusted subnet itself is the gate.* The
company network is the credential; a host on it is trusted, a host
off it can't connect. This is a real, common pattern and not
wrong by itself.

But it's safe **only if the cloud edge provably cannot route to
it.** "Authless" + "reachable from the internet" is an open door.
The discipline that keeps the boundary real:

- **Bind the LAN interface, not `0.0.0.0`.** The service listens
  on the host's LAN IP only. A `0.0.0.0` bind plus any future
  port-forward = exposed.
- **The edge must *reject* the private surface, not merely
  "not forward" it.** Passive non-forwarding is one config edit
  away from a leak. Make the cloud-fronted relay **actively
  refuse** the private namespace — e.g. the VPS-fronted store
  rejects any key under the reserved `_pkg/` prefix, so even if
  something proxies through the edge, the private packages can't
  come out. Defense in depth: the boundary is enforced at the
  edge, not just assumed at the bind.
- **One namespace, two surfaces, asymmetric exposure.** When a
  LAN service and an internet-fronted service share a backing
  store, the reserved/private keys are readable on the LAN
  surface and **hard-blocked on the edge surface**. State the
  block in code, not in a firewall rule someone can widen.
- **Authless is a property of the boundary, not the service.**
  The moment the service could be reachable off-subnet (a new
  forward, a VPN that bridges networks, a misconfigured edge),
  the "subnet is the auth" assumption is void and the service
  needs real auth. Re-validate the assumption whenever the
  network topology changes, not once at build time.

The generalisable rule: **subnet-gated authless is fine while the
gate is real and enforced at the edge; it is a vulnerability the
instant the edge can reach in.** Bind narrow, make the edge
actively reject the private surface, and treat any topology change
as invalidating the "trusted network" assumption until re-checked.
See anti-pattern #37.

<a id="reverse-proxy-subpath"></a>
### 23.2 A web app served behind a reverse-proxy subpath uses relative URLs / a configured base path

A web app reached through a reverse proxy is rarely at the origin
root — the edge mounts it under a **path prefix**
(`example.com/app/…` → `localhost:PORT/…`). An app that assumes it
lives at `/` works on `localhost` and **breaks the moment it's
proxied**: absolute links and asset paths (`/static/x.js`,
`/api/y`, `<a href="/page">`, a cookie `path=/`) resolve at the
*origin* root, not under the prefix, so assets 404, API calls miss,
and redirects escape the mount. It passes every local test and
fails only behind the edge — the deployment you didn't run.

Make the app **mount-point agnostic**:

- **Reference siblings relatively, not from root.** `./static/x.js`
  / `static/x.js`, not `/static/x.js`; build links and fetches off
  the document's own base, so they ride whatever prefix it's served
  under.
- **If you must build absolute paths, derive them from a single
  configurable base** (`BASE_PATH` / a `<base href>` / the
  framework's "root path" setting) injected at deploy time — one
  knob the proxy mount sets, not a constant baked at `/`.
- **Don't bake the host/scheme either.** Behind a TLS-terminating
  proxy the app sees plain HTTP on a private port; build external
  links from forwarded headers / config, not from what the socket
  reports, or you emit `http://…:PORT` links that leak the backend
  and break under HTTPS.
- **Verify behind the proxy, not just at `localhost:PORT`.** The
  bug is invisible at the root origin; exercise the app *through*
  the edge under its real prefix.

The generalisable rule: **an app served behind a reverse proxy
doesn't own its mount point — reference assets/links/cookies
relatively or off one configured base path, derive external URLs
from forwarded headers, and test through the edge under the real
prefix; an app hardcoded to `/` works on localhost and 404s under
the proxy.** See anti-pattern #56.

<a id="edge-new-conn-throttle"></a>
### 23.3 A shared edge throttles new connections — retry with backoff, reuse a kept-alive connection

A cloud edge / reverse proxy / bastion fronting many clients
**rate-limits or saturates on *new* connections** (SSH
`MaxStartups`, a connection cap, a relay under load). The trap: a
client that opens a **fresh** connection per operation and gives up
on the first timeout reads the edge's back-pressure as *"the peer is
**down**"* — a heartbeat marks the robot OFFLINE, a deploy /
calibration push "fails," an upload 502s — when the peer is fine and
the edge was simply busy.

**Recognise it:** SSH prints `Connection closed by UNKNOWN port 65535`
(or `Connection closed by <edge> port 22`) — the edge accepted the TCP
then dropped the session **pre-auth**. That's the edge's connection
back-pressure (`MaxStartups`), **not** the peer being down — and **not**
some other app/session co-hosted on the edge: a bastion that also fronts
a WordPress / Feishu-SSO portal / etc. does not cause this, *connection
count* does (your own retry-storms + autossh reconnects + parallel
deploys are the usual culprits). The same signature also appears when the
reverse tunnel is stale (the edge port has no backend after a rig reboot)
— retry-with-backoff covers the autossh re-establishment window; escalate
to a LAN `systemctl restart ssh-tunnel` only if it persists.

- **Retry with backoff; don't fire once.** Gate the operation on a
  reachability probe that retries with a sane timeout + backoff
  before declaring failure. A single short-timeout attempt through a
  busy edge is a false negative, not a verdict.
- **Reuse a kept-alive connection — route EVERY op through one shared
  SSH ControlMaster socket** (`-o ControlMaster=auto -o
  ControlPath=~/.ssh/ctrl/%C -o ControlPersist=10m`): the first connect
  pays the throttled new-connection cost **once**, and every later
  ssh/scp/rsync rides that socket without opening a new edge connection.
  Observed directly through a saturated VPS jump: the ControlMaster alias
  connects fine while `ControlMaster=no` (a fresh connection per op — the
  default for one-shot probes / deploys / `build-on-rig`) fails with the
  `port 65535` close every time. A tool that opens N fresh connections per
  "apply" is self-DoSing the edge — funnel them all through one mux.
  (Balance with [§6.4](engineering-rules.md#bounded-reconnect): reuse the handle, cap
  *genuine* reconnects. Caveat: a shared `-L` forward collides on a second
  forwarding invocation — mux the one-shot ops, keep a long-lived `-L`
  feed on its own unmultiplexed connection.)
- **Bake the mux into `~/.ssh/config`; reach via the alias — never
  hand-roll a per-command `ProxyCommand`.** Configure it ONCE as a
  `Host <rig>` alias (`ProxyJump <bastion>`, `ControlMaster auto`,
  `ControlPersist 4h`), and crucially give the **bastion `Host` its OWN
  `ControlMaster` too** — otherwise every `ProxyJump` dial opens a fresh
  edge connection even while the rig session is muxed. Then reach it as
  `ssh <rig>` (or one helper that `ssh -O check`s the master and opens it
  with backoff). The recurring self-inflicted failure: a human/agent
  debugging reaches the rig with an **ad-hoc**
  `ssh -o ProxyCommand="ssh -W host:port jump@edge" -p… rig@127.0.0.1`
  per command — which *bypasses* the configured alias AND opens **two**
  un-multiplexed edge connections each time, saturating `MaxStartups`
  within a handful of commands (the `port 65535` close). One alias, one
  master **per hop**, one helper — not a hand-typed `ProxyCommand`, and
  not a fresh `-W` jump per command.
- **In Unomove that "one helper" already exists — it is `uno-rig` (and
  `unojump`); never invoke raw `ssh`/`scp`/`rsync` against a rig yourself.**
  `uno-rig` (vendored at `scripts/uno-rig`, canonical source
  `unolib/bin/uno-rig`) is the canonical multiplexed-SSH transport: you pass a
  robot *ref* (UUID / alias / hostname) and it resolves the ProxyJump hop from
  the registry ([§23.4](#reach-a-fleet-host-via-its-registry)) and funnels the
  op through the shared ControlMaster socket above. `unojump` is its jump-hop
  companion. Route every rig command/copy through them — in Python via
  `unogui.app.unorig` (or `unogui.app.ssh.run_ssh`, which already prefers
  uno-rig), never a hand-assembled `ssh` line. Hand-rolling raw `ssh` bypasses
  the mux (self-DoSing the edge — the failure right above), skips registry ref
  resolution, and hardcodes a hop that DHCP churn will invalidate. This holds
  for the control/command path too: recording start/stop, `unopilot`, gear, and
  relaunch go through the uno-rig transport, not a bespoke `ssh` call. (When
  scripting by hand, the configured `ssh <alias>` IS this muxed path; if the
  `uno-rig` wrapper falsely reports "tunnel down" while the alias works, trust
  the alias and confirm liveness via the registry heartbeat, §23.4.)
- **Size the timeout to the edge, not the LAN.** A timeout tuned for
  a direct link is too tight through a saturated relay — size it to
  the worst-case path ([§9.9](engineering-rules.md#shared-resource-contention)).
- **If you *own* the edge, size its new-connection cap to the fleet.**
  The mitigations above are for an edge you *don't* control; when the
  bastion/relay is yours, also raise its limit to fit the real peak —
  `sshd`'s `MaxStartups` (the unauthenticated-connection cap) and the
  listen backlog on a host fronting N rigs' heartbeats + deploys. A
  default `MaxStartups 10:30:100` throttles a modest fleet; set it to the
  measured peak. Client backoff/mux and server headroom are complementary
  — do both, don't rely on one.
- **Default writes to the key-based SSH transport, not HTTPS-through-the-
  proxy.** When the host is reached over a flaky HTTPS proxy/edge (a
  corporate proxy, a fake-IP tunnel — Git's `198.18.x.x` symptom),
  default the **push** — the write that must converge
  ([§23.5](#reverify-before-retry)) — to **key-auth SSH on a
  proxy-friendly port (443)**, multiplexed as above; key auth + one mux
  beats an HTTPS handshake per push through a dropping proxy. **Pin it
  without rewriting every remote:** Git's
  `url."ssh://git@ssh.github.com:443/".pushInsteadOf "https://github.com/"`
  makes every HTTPS remote *push* over SSH-443 while *fetch* stays HTTPS
  (so a `pushInsteadOf` is a one-line, fleet-wide default, not a
  per-repo `set-url`). New repos can just take an SSH-443 origin outright.

The generalisable rule: **a shared edge throttles new connections,
so an operation through it retries with backoff and rides a
kept-alive connection — a one-shot, short-timeout new connection
false-reads edge back-pressure as a dead peer; and behind a flaky
HTTPS proxy/edge, default writes to the key-based SSH transport on a
proxy-friendly port (pinned via `pushInsteadOf`, multiplexed), not an
HTTPS handshake per push.** See anti-pattern #68.

<a id="reach-a-fleet-host-via-its-registry"></a>
### 23.4 To reach a fleet host by IP, ask the registry first — never hardcode or scan

A fleet host on DHCP **does not own its address** — the lease moves,
and the host's *old* IP gets handed to some unrelated phone or laptop.
So the way to reach a rig is **not** a remembered IP and **not** a
subnet scan; it's the **self-reported address in the registry**, which
the host pushes on every heartbeat.

The lookup order, every time:

1. **Registry heartbeat `info.lan_ip`** — `GET /tailnet/clients`
   (bearer = the enroll token) returns each host's row; `lan_ip` is the
   address it last reported about *itself*. Authoritative and current
   (also carries `last_seen` — stale heartbeat = the host is the
   problem, before you blame the path). Use this **first**.
2. **The reverse tunnel / cloud edge** — the host's outbound autossh to
   a fixed edge port. IP-independent, so it survives the DHCP churn; the
   fallback when you're off the host's LAN.
3. **A direct LAN connection to the IP from step 1** — only *after* the
   registry gave it to you. Lower latency for big transfers.

A **subnet ping-sweep is a last resort, not a first move**: it's slow,
it can't tell the rig from the desktop that inherited its old IP, and
it fails silently when the host is on another segment. Reaching for it
means you skipped the registry.

Two corollaries this depends on:

- **The heartbeat MUST report `lan_ip` — on *every* lifecycle stage,
  not just factory.** If the onroad heartbeat drops the field (or a host
  runs a heartbeat binary that predates it), its registry row shows an
  empty `lan_ip` and step 1 silently falls through to the scan you were
  trying to avoid. The field reporting is one shared code path the
  factory and onroad heartbeats both use — not a factory-only nicety.
- **An empty `lan_ip` with a fresh `last_seen` is a heartbeat bug;** an
  empty `lan_ip` with a stale `last_seen` means the host is down — reach
  for the power-cycle/escalation, not a wider scan. Read both fields.
- **Bulk data between two co-located hosts takes the LAN path, not the WAN
  edge.** Reaching a host is steps 1–3 above; *moving big data* to it has
  the same answer for a stronger reason. When the registry `lan_ip` shows
  the peer on **your** LAN, PUT/GET bulk transfers **straight to it at LAN
  speed** — do **not** detour them through the thin metered cloud edge (a
  rig → WAN-up → edge → tunnel-down → peer round trip starves and **502s**
  multi-MB files while a tiny probe passes,
  [§9.9](engineering-rules.md#shared-resource-contention)). Fall back to
  the relay only when the peer is genuinely off-LAN, and derive the target
  from the registry, never a new env var
  ([§7.5](engineering-rules.md#env-var-config)). See anti-pattern #124.

The generalisable rule: **a host's address lives in the registry it
heartbeats to, keyed on its stable id — query that (and its freshness)
to reach it, and route bulk transfers to a co-located peer over its
`lan_ip` rather than the metered WAN edge; a hardcoded IP rots the moment
DHCP moves the lease, and a scan can't distinguish the host from whatever
took its old address.** See anti-patterns #76, #124.

<a id="reverify-before-retry"></a>
### 23.5 A transport drop mid-operation is not proof the write failed — re-verify the end-state before retrying

A networked write — a `git push`, a deploy, an upload, an API / DB
commit — that errors with **"connection closed" / "broken pipe" / a
sideband disconnect / a read timeout** has **not** necessarily failed.
On a flaky or throttled link ([§23.3](#edge-new-conn-throttle)) the
server frequently **applied the change and then the channel dropped
before the ack came back** — the work is done, only the confirmation is
missing. (And the *verify* step is on the same flaky link, so a failed
re-check can itself be a false negative.) Retrying blind then wastes a
round-trip, or — if the op isn't idempotent — **duplicates** it.

We hit this repeatedly: a `git push` reported `Connection closed …
sideband packet … remote end hung up`, but the ref had already
advanced — the retry printed **"Everything up-to-date."** And a
post-push verify `fetch` dropped on the same link, which *looked* like
the push had failed when it had landed.

- **Re-verify the end-state, not the channel's report.** Before
  retrying, read the consumer-visible state cheaply and idempotently —
  `git fetch` then `local == remote`, a `GET` of the resource, the row's
  current value — and retry only the part that genuinely didn't take.
  This is anti-pattern #88's "gate on the published state" applied at the
  moment of retry.
- **Make the retry idempotent so a redo is harmless.** Upsert by a
  stable id ([§25.2](#idempotent-writes)), `push` (a no-op when already
  there), `PUT` not `POST`. Then "did it really fail?" stops mattering —
  re-running converges either way.
- **Don't conclude "peer offline / op failed" from one dropped
  connection.** The verify channel is as flaky as the write channel
  ([§23.3](#edge-new-conn-throttle)) — re-probe with backoff before you
  act on a failure verdict, exactly as for a throttled new connection.

The generalisable rule: **a transport error mid-operation (a dropped
connection, a lost ack, a verify that timed out) is not proof the write
failed — the server may have applied it and lost the channel; re-verify
the consumer-visible end-state with an idempotent read before retrying,
and make the write itself idempotent so a redo converges instead of
duplicating.** See anti-pattern #91.

---

<a id="when-in-doubt"></a>
## 24. When in doubt

> **When this fires** — the change crosses repos you can't see, a constraint is unwritten, or two non-obvious paths exist.
>
> **Do** — **stop and ask** — a clarifying question is cheaper than a wrong guess.

If a question crosses repos you can't see, if a constraint isn't
written down, if the choice between two paths is non-obvious —
**say so to the user**. They can answer in seconds. Don't guess.

A guess that turns out wrong costs the user more than a clarifying
question ever does.

---

<a id="external-integration"></a>
## 25. Integrating with an external service / API

> **When this fires** — a two-way sync, any write to an external service, an event / webhook integration, or OCR / LLM field extraction.
>
> **Do** — a periodic **reconcile poll** is the correctness floor (push is latency only); idempotent upserts keyed on a stable id; read the most structured source, validate the lossy one.

Code that talks to a service you don't control — a SaaS API
(calendar, docs, chat, OCR), a cloud provider, a partner endpoint —
fails in ways in-process code doesn't: events get dropped, writes
get retried, tokens expire, the far end rate-limits or returns a
*different shape* next quarter. Treat the boundary as untrusted and
the far state as something you **reconcile to**, never assume.

<a id="reconcile-poll"></a>
### 25.1 Event push is an optimisation; a periodic reconcile poll is the guarantee

A webhook / websocket / change-feed that pushes "something changed"
is great for **latency** and, on its own, useless for
**correctness** — events get dropped, arrive out of order, are
missed while you were restarting, or are never sent for some class
of change. Build the sync as a **periodic full reconcile poll** (the
correctness floor) **plus** an event push (the latency optimisation
on top), never the push alone:

- The poll runs on a timer (seconds to a minute, by how fresh it
  must be) and converges both sides regardless of any event.
- The push only *triggers an early reconcile* — it doesn't carry
  the sole copy of the truth. Miss the event and the next poll
  still fixes it.
- A system whose correctness depends on event delivery has no
  floor: the day the socket silently drops, it goes stale with no
  error. The poll is what turns a missed event into a latency blip
  instead of permanent drift.

See anti-pattern #55.

<a id="idempotent-writes"></a>
### 25.2 External writes are idempotent and reconcile by stable ID — a failed op must not duplicate

Every write to an external system can be retried — by you on error,
by a reconcile pass (§25.1), by the user double-clicking. So a write
must be **idempotent**, keyed on a **stable identity**, or you get
duplicates and runaway loops:

- **Match by a stable key before you create.** Carry your own id (or
  a deterministic content hash) on the remote record and **look it
  up first** — create only if absent, else update. "Create a row"
  with no duplicate check makes every retry a new row.
- **A failed op must not feed a create loop.** The trap we hit: a
  *delete* on one side failed, the next reconcile saw the record
  "still there," re-created its peer, and looped — runaway
  duplication from one failed call. Make each step idempotent *and*
  make a failure leave state where the retry **converges**, not
  multiplies. Cap / loudly log a reconcile that keeps finding the
  same diff ([§10.2 capped recovery](engineering-rules.md#capped-recovery)).
- **Key deletes and updates on the same stable identity** — never on
  a mutable field (title, name) the user can change on either side,
  or you orphan the old and create a new one.

See anti-pattern #54.

<a id="structured-over-ocr"></a>
### 25.3 Read the most structured source; OCR / LLM extraction is a lossy fallback you validate

When you need fields out of a document, take them from the **most
structured representation available** and reach for OCR / an LLM
only when there isn't one:

- A machine-readable form — an e-invoice's embedded PDF text, an API
  field, an attached XML/JSON — is exact; parse it directly. Running
  OCR/vision over the *rendered* version of data you could have read
  structured throws away precision for nothing.
- OCR / LLM output is a **guess** — validate before you trust it:
  parse amounts/dates into typed values and sanity-check (currency
  present, total = sum of lines, date in range), cluster noisy
  fields, and flag / fall back on low confidence. A "safer amount
  parse" is not polish — an OCR'd `1,234.00` mis-read as `123400`
  writes a wrong number into a system of record.
- **Pull each field from its labelled canonical slot, not a scan of
  the whole document.** A regex/LLM sweep over all the text picks up
  **decoys** — a `¥` amount buried in a remarks line, a date in a
  footer — and a stray match silently wins. Anchor to the field that
  *means* the value (an invoice's "amount in words / tax-inclusive
  total", a named key), and prefer it over any looser match.
- Layer engines by cost and structure: structured-read → cheap /
  offline OCR → cloud OCR/LLM, with backoff — so the common case is
  exact and free and the fallback is bounded.

The generalisable rule of §25: **an external boundary is reconciled,
not trusted — poll to converge (push is only latency), write
idempotently keyed on a stable id (a failed op converges, never
duplicates), and extract from the most structured source while
validating any OCR/LLM guess.** Test these paths **offline** by
asserting the *request shapes* you send (record/replay), so the
suite runs without live credentials ([§18](#tests)). See
anti-patterns #54 and #55.

---

<a id="trust-boundary"></a>
## 26. Handling money or privileged actions: the server is the trust boundary

> **When this fires** — a money / authorization value arrives from the client, a high-impact irreversible action, or a public / no-login demo.
>
> **Do** — recompute money & authz server-side from authoritative data; **dual control** + an audit trail for irreversible actions; isolate a demo from prod by construction.

When an app handles money, PII, or actions only some users may take
— a reimbursement, a payout, a role grant, a destructive admin op —
the **client is untrusted** and the **server is the only
authority.** A robotics stack rarely needs this; a multi-user web
app with real money does, and a few controls cover most of it.

<a id="server-authority"></a>
### 26.1 The client is untrusted — authorize and recompute on the server

Anything the client sends is a *request*, not a fact:

- **Never trust a client-sent value that carries money or
  authorization meaning — recompute it server-side from
  authoritative data.** An amount, price, discount, total, role, or
  target id posted by the browser can be tampered with; bind it to
  the server's own record (look up the invoice's amount, the user's
  role) and ignore the posted one — otherwise the client sets its
  own price. This is the *write*-side of [§8.22](ui-rules.md#masking)'s
  "hiding in the UI ≠ access control": there the API must not
  *return* what a role can't see; here it must not *accept* what the
  client shouldn't set.
- **Authorize every privileged action on the server**, keyed on the
  **authenticated identity** (a stable opaque id, not a display
  name the user can change) — never on a hidden form field or "the
  UI wouldn't show the button."
- **Escape user-supplied content where you render it** (XSS), and
  rate-limit endpoints that cost money or send messages. Standard,
  but only the server can enforce them.

The generalisable rule: **the client states intent; the server
decides and recomputes. Never trust a posted value that carries
authorization or money — bind it to authoritative server-side data,
keyed on a stable identity.** See anti-pattern #60.

<a id="dual-control"></a>
### 26.2 High-impact or irreversible actions take a second authority and an audit trail

For an action that moves money, deletes data, or can't be undone, a
single click by a single person is too much authority:

- **Separation of duties (two-man rule).** The requester is not the
  approver — a payout, a bulk delete, a role grant needs a *second*
  independent person to confirm. (The multi-actor form of
  [§3.1](engineering-rules.md#safety-critical)'s "no one secondary surface holds full
  control" and [§19](#reversibility)'s "ask before a destructive
  op.")
- **Cap the blast radius.** An enforced limit — a per-transaction
  amount cap, a max number of rows a purge will touch — that a
  larger action must exceed deliberately, with extra approval; never
  an unbounded "execute."
- **Audit trail.** Record who did what, when, to which record, in a
  tamper-evident log you can export. For money and irreversible ops
  it isn't optional. (An internal destructive CLI tool is the same
  shape: dry-run default, an explicit ack flag + a second
  confirmation, an audit log + notification.)

The generalisable rule: **a money-moving or irreversible action
needs a second authority, a bounded blast radius, and an audit
trail — never one click by one person with no record.** See
anti-pattern #60.

<a id="demo-isolation"></a>
### 26.3 A public / unauthenticated demo is isolated from production by construction

A demo, trial, or sandbox you expose **without login** — anyone on
the internet can poke it — must be unable to touch production. Not
"gated off by a flag": **isolated by construction.**

- **Separate deployment, mock or seeded data.** The demo runs its
  own instance against throwaway data, with **no production
  credentials, tokens, or network reach** to the real backend. A
  bug, an injected request, or a curious user can't read or write
  real records because the path isn't there to walk.
- **Isolation, not a runtime toggle — the deliberate exception to
  [§9.6](engineering-rules.md#build-backend-switch).** Elsewhere a tier is a runtime
  role in one bundle; here the safety property is *"can never reach
  prod,"* and a flag inside the prod app is one bug from violating
  it. When a gate's failure is catastrophic (a public surface
  reaching real money / PII), isolate the deployment instead of
  trusting the flag.
- **No real secrets in a world-reachable bundle.** A public demo
  ships zero prod keys; an upstream it needs is a mock/stub, not the
  live service.

The generalisable rule: **a publicly-reachable, unauthenticated
demo is isolated from production by construction — its own
deployment, mock/seed data, no prod credentials or network path —
because for a public surface "cannot reach prod" must hold even if
a gate fails.** See anti-pattern #61.

<a id="per-identity-access"></a>
### 26.4 Grant access by the requester's own public key on a least-privilege account — never a shared secret

When you give a person or peer access to a host or service, authorize
**their own public key** on a **least-privilege account**, gated by an
approval step ([§26.2](#dual-control)). **Never hand out a shared
password or a copied private key.** A shared secret fails three ways
that a per-identity key doesn't:

- **It can't be revoked for one person.** When an operator leaves or a
  laptop is lost, you must rotate the secret for *everyone* and
  re-distribute — so in practice nobody does, and stale access lingers.
  A public key is removed from one `authorized_keys` line; everyone
  else is untouched.
- **It loses attribution.** A shared credential can't answer "who did
  this?" — the audit trail ([§26.2](#dual-control)) collapses to one
  identity. Per-key access logs the actor.
- **It maximises blast radius.** One leak of a shared secret is total
  compromise; the secret also has to *travel* (a DM, a paste, a second
  machine) — each hop a new exposure. A public key is public: nothing
  secret is ever transmitted.

Scope the account to **exactly what the role needs** — a
forwarding-only jump account (no shell, no sudo), a read-only role —
so a compromised key can't exceed the grant. The provisioning itself
goes through an approval gate (the requester submits *their* pubkey; an
admin, or a self-service flow with a scoped enroll token, authorizes
it) — recording who was granted what, when.

The generalisable rule: **provision access by authorizing the
requester's own public key on a least-privilege account behind an
approval gate — never distribute a shared password or private key; a
per-identity credential revokes and audits individually and transmits
no secret, while a shared secret can't be revoked for one person,
loses attribution, and forces a fleet-wide rotation on every
departure or leak.** See anti-pattern #85.

<a id="fleet-access-ca"></a>
### 26.5 At fleet scale, trust a certificate authority — not a key on every host

[§26.4](#per-identity-access) gets the *credential* right (a
per-identity key), but its naive placement doesn't scale: appending
each identity's key to every host's `authorized_keys` is
**O(identities × hosts)**. A new operator must be provisioned on
every host, the per-host lists **drift** (some hosts get missed),
and revoking one identity means editing **every** host — so in
practice nobody does, and stale access lingers across the fleet.
Past a handful of hosts this is where access stops being *managed*
and becomes copy-paste.

The scaling fix keeps §26.4's per-identity property but moves the
trust to a **certificate authority**:

- A central CA **signs short-lived SSH user certificates** for an
  enrolled identity (which still submits *its own* pubkey through the
  approval gate, [§26.2](#dual-control)/[§26.4](#per-identity-access)).
- Every host **trusts the CA** (`TrustedUserCAKeys`), not each
  identity. A new identity enrolls **once** with the CA and
  immediately reaches **every** host — no per-host distribution, no
  drift.
- Certs are **short-lived** (hours/days), so a leaked cert expires on
  its own — the credential is time-bounded, not forever.
- Revocation is **central and instant** via a key-revocation list
  (KRL) the hosts load — one entry kills an identity fleet-wide, vs
  editing N `authorized_keys`.
- Harden the now-uniform front door (brute-force rate-limit /
  fail2ban on the exposed port).

The trade-off: the CA private key is a **fleet-wide secret** (its
compromise is total), so it lives on a hardened host, signs only
through the audited approval flow, and is itself rotated on a
schedule.

The generalisable rule: **per-identity keys are right, but
`authorized_keys`-per-host is O(identities × hosts), drifts, and
revokes per-host — at fleet scale have every host trust a central CA
(`TrustedUserCAKeys`) that issues short-lived per-identity
certificates: an identity enrolls once and reaches every host, certs
expire on their own, and a KRL revokes one identity fleet-wide
instantly; guard the CA key as a fleet-wide secret.** See
anti-pattern #114.

<a id="encode-untrusted"></a>
### 26.6 Encode untrusted data for its sink — escape on output, parameterize on input

[§26.1](#server-authority) keeps the client from forging *authority*;
this keeps untrusted *content* from being executed. The bug is one
shape with many faces: a value that originated outside your trust
boundary (a form field, a job name, a filename, a device-reported
string, a row another user wrote) is **interpolated raw into a
string an interpreter then parses** — HTML/JS (→ **XSS**), a SQL
query (→ **SQL injection**), a shell command (→ **command
injection**), an LDAP/template/log line. The interpreter can't tell
your structure from the attacker's payload, so the data becomes
code: a stored job name like `<script>…</script>` runs in every
viewer's session; a name like `'; DROP TABLE …` runs in your DB.

- **Output: encode for the exact context at render time.** Don't
  build HTML by string concatenation — use a templating engine that
  **auto-escapes**, and escape for the *specific* sink (HTML body vs
  attribute vs JS vs URL are different encodings). The same value is
  safe in one context and live in another, so encode where you emit,
  not where you store.
- **Input: parameterize, never interpolate.** Use **bound query
  parameters** (prepared statements), an **argv array** for a
  subprocess (never `shell=True` / a concatenated command line), a
  real parser for the format — so untrusted text is always *data* to
  the interpreter, never syntax. Validate/allowlist on top
  (length, charset, a known set) as defence in depth, but
  parameterization is the actual fix.
- **It's a server-side / boundary control.** Client-side escaping is
  cosmetic ([§26.1](#server-authority)); the encoding must happen on
  the trusted side that feeds the interpreter. A field that's
  *stored* raw is fine — the danger is at the **sink**, so every sink
  encodes for itself.
- **Treat stored data as untrusted too** (stored XSS): content that
  was clean when written can be rendered into a new context later;
  the renderer still encodes.

The generalisable rule: **untrusted data that crosses into an
interpreter (HTML/JS, SQL, a shell, a template) must be encoded for
that sink — auto-escape on output in the right context, parameterize
(bound params / argv array) on input — never interpolated raw, or the
data becomes code (XSS, SQL/command injection); encode at the sink on
the server, and treat stored values as untrusted when you render
them.** See anti-pattern #118.

---

<a id="build-fix"></a>
## 27. Fixing a build systematically

> **When this fires** — a build is failing, behaving differently across hosts, or green-but-wrong.
>
> **Do** — method, not flag-poking: reproduce from a clean committed tree → pin every input → localise the layer → build for the target → verify on the **deployed** form.

A build that fails — or worse, *succeeds and ships the wrong thing*
— is fixed by **method, not by poking flags until it goes green.**
The same ordered checklist resolves nearly every build issue this
stack hits; it's [diagnostic-first (§15.1)](#diagnostic-first) plus
the reproducible-build basics (one source ref, locked inputs,
isolated layers, verified on the deployed form). Each step below is
its own rule — this section is the **order to apply them**.

1. **Reproduce it deterministically, from a clean committed tree.**
   Get the exact failing command and run it from a **committed**
   source ref on a **clean** tree — a build compiles the working
   *tree*, not git ([§20.1](#merge-to-main)), so uncommitted /
   collaborator WIP and a `-dirty` stamp ([§12.1](#build-version-stamp))
   make the failure unreproducible. Can't reproduce it clean? You're
   debugging the tree, not the build.

2. **Pin every input; remove ambient state.** A reproducible build
   depends only on *declared* inputs: deps locked (`uv.lock`,
   [§21](#python-env)); the toolchain / backend an explicit declared
   variable, not an `os.environ` read
   ([§9.6](engineering-rules.md#build-backend-switch), [§7.5](engineering-rules.md#env-var-config)); the
   source ref fixed. If two hosts build differently, **diff the
   inputs** (lock, toolchain version, the build stamp) before
   reading code — the
   [authoritative-vs-actual](anti-patterns.md#recurring-shapes) gap
   in build form.

3. **Localise the failing layer — the error names it.** Source
   *compile* → *link* (a wrong-arch / wrong-libc shared lib,
   [§11.1](#platform-libs)) → *package / freeze* (a freezer silently
   dropping a module whose native dep is absent,
   [§11.2](#freeze-native-deps)) → *deploy / runtime layout* (a
   compiled / installed tree has no source, no `-m`, no dev paths,
   [§13.1](#dev-vs-deployed-layout)). Fix the layer that broke, not
   the one that surfaced the symptom.

4. **Build for the target, on a safe host.** Cross-build with a
   toolchain that targets the right `(os, arch, libc)`
   ([§11](#cross-compile)); don't heavy-build on a live shared rig —
   a compile starves a latency-critical peer through memory
   bandwidth `nice` can't isolate
   ([§9.9](engineering-rules.md#shared-resource-contention)) — build stack-stopped or
   off-box.

5. **Verify on the *deployed / production-built* form, not the dev
   one.** "It compiled" ≠ "it runs there": import the entry points
   of the frozen artifact ([§11.2](#freeze-native-deps)), exercise
   the installed / compiled layout ([§13.1](#dev-vs-deployed-layout)),
   and confirm the build stamp on the target is the sha you meant
   ([§12.1](#build-version-stamp)). A green build is a hypothesis
   until the artifact runs on the target.

6. **Record the fix.** A build fix with a non-obvious cause (a
   missing native dep, a host-arch clobber, a backend flag) goes in
   the decision record ([§18.3](#decision-record)) — otherwise the
   next person re-breaks it.

The generalisable rule: **fix a build by method, not by flag-poking
— reproduce from a clean committed tree, pin every input, localise
the failing layer (compile → link → package → deploy), build for the
target off a live box, and verify on the deployed form; a build that
went green by guessing is unreproducible and ships the wrong
thing.** See anti-pattern #67.

---

