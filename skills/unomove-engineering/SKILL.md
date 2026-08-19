---
name: unomove-engineering
description: Use for Unomove code changes touching interfaces, message/API fields, schemas, units, safety-critical paths, controls, sensors, runtime processes, pub/sub, config/defaults, env vars, language choice, shared reusable logic, or unolib.
---

# Unomove engineering rules

Load this skill before changing backend/runtime behavior or shared
engineering contracts.

## Read

1. Start with `../../quickref.md` if you need to locate the exact rule quickly.
2. Read the relevant section of `../../engineering-rules.md`.
3. If the change also involves shipping, tests, deployment, repo hygiene,
   network, or external services, load `../unomove-delivery/SKILL.md`.
4. If it touches UI, load `../unomove-ui/SKILL.md`.

## Always apply

- Schemas are append-only; do not silently renumber, reorder, delete, or narrow
  existing fields.
- Use SI units unless the field name explicitly carries a non-SI unit.
- Safety-critical paths need one authority, explicit freshness, loud fallback,
  and verification on the target when practical.
- Do not add env-var behavior/config/device switches. Use hardware/role-derived
  logic or checked-in config; ask if the choice has no proper home.
- Prefer Rust for new reusable code. Shared logic belongs in `unolib`, not a
  per-repo copy.
