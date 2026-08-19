---
name: unomove-ui
description: Use for any Unomove UI, frontend, GUI, operator surface, layout, theme, visual restyle, control widget, status display, masking, or telemetry visualization change.
---

# Unomove UI rules

Load this skill before changing any interactive surface.

## Read

1. Read the relevant part of `../../ui-rules.md`.
2. For a fast map, scan `../../quickref.md` for `§8`.
3. If the UI action writes to a backend, also read the relevant authority,
   safety, schema, or stale-value rule in `../../engineering-rules.md`.
4. Before done, load `../unomove-review/SKILL.md`.

## Always apply

- Render verified wire/backend state, not optimistic local input.
- Confirm control actions by reading back the authority's actual state.
- Layout fixes must be verified cold, with caches wiped or cache keys bumped
  when stale layout can persist.
- Mask sensitive infrastructure, PII, tokens, and money details on
  always-visible surfaces.
- New UI follows the house design language and uses shared theme tokens.
