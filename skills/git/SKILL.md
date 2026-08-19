---
name: git
description: Use when cloning/pushing/creating a repo — SSH-key origin + `~/unocoding/setup.sh`, never `gh` (not authenticated on these hosts).
---

## Git over the SSH key, not `gh`
Prefer the key-based SSH origin (`ssh://git@ssh.github.com:443/...`) for clone/fetch/push — `gh` is NOT authenticated on these hosts and is not the default. For a NEW repo, wire it with `~/unocoding/setup.sh <path>` (younuozhixing SSH origin + vendored rules + bump.sh registration); don't reach for `gh repo create`. Keep commits surgical; fast-forward + push (§20 repo hygiene).
