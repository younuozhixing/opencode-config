---
name: machine-access
description: Use the system-level unoai-jump tool for company-machine commands, diagnostics, file transfers, and tunnels from any repository.
---

## Shared System Tool

Use `unoai-jump` for company-machine access from every repository, instead of raw
SSH/SCP or creating a separate bastion connection. It is a standalone system
tool, not a dependency on the UnoAI app, unotoken, or a project's source tree.
Do not copy its implementation into another repository.

Find the installed command with `command -v unoai-jump` and verify its identity
with `unoai-jump --version`. The standard entry is
`~/.local/bin/unoai-jump`; prefer that explicit entry if a project's PATH shadows
it with an older bundled copy. A bundled app may intentionally pin its own version.
For a missing installation, use an available trusted source checkout:
`python3 ~/unoai/bin/unoai-jump --install`. This installs a checksum-addressed
release independently of the checkout. Do not clone or invent another copy just
to run a remote command. Another machine needs its own installation and existing
authorized SSH configuration; never copy private keys as part of tool rollout.

## Use

- `unoai-jump --list` lists configured machine names; `--why <machine>` explains routing.
- `unoai-jump --read-only --timeout 30 <machine> 'cat /etc/hostname'` runs diagnostics.
- `unoai-jump --progress --timeout 120 <machine> '<authorized command>'` streams output.
- `unoai-jump --json <machine> '<command>'` returns structured results for automation.
- `unoai-jump --progress sync ./build/ <machine>:/tmp/staged-build/` transfers files.
- `unoai-jump sync <machine>:/tmp/report ./report` downloads a file.
- `unoai-jump forward <machine> -L 127.0.0.1:18090:127.0.0.1:8090` opens a local tunnel;
  use `cancel` with the same specification to close only that tunnel.

Global flags go before the machine/subcommand. Use `--read-only` for inspection;
the default is write-enabled, but **capability is not authorization**. Only mutate
the machines/files/services the user's current task covers. Broad access does not
authorize actuator movement, service restarts, account changes, or wider deployment.

Writes are not automatically replayed. On `outcome_unknown`, verify the remote
end-state before retrying. Timeout means the local invocation ended, not proof the
remote command stopped. JSON retains bounded output tails; check truncation flags.
Sync uses rsync checksums and a separate partial directory; rerun after inspecting
a failure to resume. Stage deployments, verify them, then use the project's normal
activation workflow. Directory sync is not an atomic release.

Never kill shared target/bastion masters to recover one call. Use the tool's
bounded recovery and preserve unrelated connections. Missing authorization or an
unreachable target must be reported, not bypassed with another transport.
