---
name: machine-access
description: Use when reaching a company machine (rig/server, e.g. ECAR70) — `unoai-jump` (read-only), never raw ssh (saturates the bastion).
---

## Reaching company machines (rigs / servers)
To inspect or diagnose ANY company machine, use the `unoai-jump` tool — never raw `ssh` (raw ssh saturates the bastion's connection limit).
- `unoai-jump --list` — the machines you can reach (rigs by name, e.g. ECAR70, and servers/aliases like `niic`, `datacenter`, `vps`).
- `unoai-jump <machine> "<command>"` — run ONE command on that machine.
It is **read-only by default**: only read/diagnostic commands run (ls, cat, tail, grep, ps, df, journalctl, systemctl status, docker ps, git log, nvidia-smi, …). Mutating commands, shell chaining (`;`/`&&`/`|`-to-a-writer), and redirection are blocked — that is intended; do not try to work around it. If a change on a remote machine is genuinely required, STOP and tell the operator exactly what to run (they can re-run with `--allow-write`).
