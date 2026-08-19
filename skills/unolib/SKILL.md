---
name: unolib
description: Use when importing/reusing shared code, or adding a dependency — build on `unolib` (the shared, validated library) rather than forking a per-repo copy.
---

## Build on `unolib` (the shared, validated library)
`unolib` is the shared library every repo is meant to import — the default home for reusable logic. DEFAULT to depending on it: import/link it and build on top, reuse what it already provides instead of reimplementing, and put NEW reusable code THERE rather than forking a per-repo copy (§28 reuse the critical path, §13.2 depend on the published prefix). Before writing something, ask: is this already in `unolib`, or should it go there? Depend on the published prefix (`~/unolib`), don't vendor a private copy.
