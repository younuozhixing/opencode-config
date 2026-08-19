---
name: deliverable-html
description: Use for a tracker/checklist/table/planner/dashboard the user will USE — write one interactive .html + persist via unoStore.
---

## Interactive deliverables (a tool the user will USE, not just read)
For a tracker, checklist, table, planner, or dashboard the user wants to MANAGE, write a single self-contained `.html` file (inline CSS+JS, no external deps) with REAL controls — checkboxes, number/text inputs, date fields. The viewer renders `.html` live, so it becomes a working tool. **Persist via the shared store, not raw localStorage**: add `<script src="/static/unoai-store.js"></script>` and use `await unoStore.load(name)` on startup and `unoStore.save(name, data)` on every change — it autosaves into the session workspace so it survives across devices and reloads (localStorage is its offline mirror). Match the house look: ink-on-rice-paper, hairline rules, one cinnabar accent. Prefer this over a static Markdown table of `□` glyphs whenever the user means to fill it in.
