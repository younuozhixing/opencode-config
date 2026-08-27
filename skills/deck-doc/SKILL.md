---
name: deck-doc
description: Use when building a BINARY Office file — a PowerPoint (.pptx), Word (.docx), or Excel (.xlsx) slide deck / document / spreadsheet. Not for HTML, not for PDF via soffice, not for plain-text manuals.
---

## Producing a file deliverable (.pptx / .docx / .xlsx / .pdf)
USE THE SKILL. A tested pptx/docx/xlsx/pdf skill is installed — it builds decks with pptxgenjs and renders a thumbnail to verify. Do NOT hand-roll a long python-pptx script with your own helper functions: one typo (a wrong kwarg) crashes the whole build before it saves, so you get NO file. Let the skill do it.
If you must build a file programmatically anyway, write it to disk EARLY and keep saving — never leave the only copy inside an unfinished build script.
- Build it with a SINGLE runnable script that ends by saving (e.g. python-pptx `prs.save("out.pptx")`), and RUN it as soon as a few slides/pages exist so a real file appears — then extend the script and re-run to overwrite. Do not construct a 20-slide deck across many edits and save only at the very end: if the turn is interrupted there, the whole deck is lost.
- For a large deliverable, `save()` incrementally (after each section) so a partial-but-real file is always on disk.
- REGENERATING a deliverable that already exists: keep the OLD file until the new one is fully built, then swap it in atomically — build to a temp path and `os.replace(tmp, final)` (or save under a tmp name, then rename). NEVER delete or truncate the existing deliverable first: if the rebuild then fails or is interrupted, the operator is left with NO file (the panel goes empty). Overwrite the SAME canonical filename in place — don't scatter _v2/_final copies; atomic-replace just means the previous good version survives a broken rebuild.
- Before you report done, VERIFY the file exists and is non-trivial (e.g. `ls -la out.pptx`, reopen it) — "the script is written" is not "the file is saved". The operator downloads the FILE, not the script.

## Branding — every deliverable carries the 优诺智行 logo
Every file you hand the operator (deck, document, sheet, PDF) is a COMPANY deliverable and MUST carry the 优诺智行 (Younuo Zhixing) mark + a trademark line. Transparent-PNG assets are installed at `~/.claude/skills/uno-deck/assets/brand/` (`younuo-logo-horizontal.png`, `younuo-logo-stacked.png`, `younuo-mark.png`) — copy the one you need into your build dir first (node/pptxgenjs won't expand `~`).
- **Decks (.pptx):** follow the `uno-deck` skill — full logo on the cover, the mark-only symbol small in a content-slide corner, `© 优诺智行 Younuo Zhixing` in the footer.
- **Documents (.docx) / PDF:** the horizontal logo small in the header (or on the title/first page) + `© 优诺智行 Younuo Zhixing` in the footer.
- **Sheets (.xlsx):** the trademark line in a header/footer or title cell; embed the logo where the format supports it.
Keep it tasteful and unobtrusive — it marks ownership, it doesn't decorate. Never stretch or recolour the logo; preserve its aspect ratio.

## 中文排版 — Chinese/CJK text in a deck (MUST follow, or you get 断字/tofu)
A DrawingML run has THREE font slots: Latin `<a:latin>`, East-Asian `<a:ea>`, complex `<a:cs>`. python-pptx's `font.name` and most pptxgenjs code set ONLY the Latin slot, so Chinese characters fall back to the theme's Latin font (e.g. Liter/Inter) which has NO CJK glyphs → 断字 / tofu / broken characters. So whenever a deck has ANY Chinese:
- Use a REAL CJK font for ALL text: **MiSans** (installed) or Source Han Sans SC / Noto Sans CJK SC. NEVER put Chinese in a Latin-only display font (Liter, Inter, Poppins…).
- Set the EAST-ASIAN slot, not just Latin. In pptxgenjs pass the CJK font as the run `fontFace`; in python-pptx `font.name=` is not enough — also set `<a:ea typeface=...>` on the run's rPr (or just run the repair step below, which does it for the whole deck).
- Turn WORD-WRAP ON for every text box (`word_wrap=True` / wrap='square') and don't pack columns with manual spaces/tabs — use a real TABLE or separate placeholders, so Chinese lines wrap at the box and stay aligned (不连贯/错位 comes from wrap='none' + space-columns).
- NO stray whitespace in 中文: Chinese uses no spaces between characters, so never emit a leading indent (`  云端`), trailing space, double space, or an ideographic space (　) inside Chinese text — those show as an 'unknown white space' gap. A single space around a `·` or `/` separator (`AI 在环 · 世界模型`) is fine; a space between two 汉字 is a bug.
- UNIFORM 中↔英 spacing (盘古之白): put EXACTLY ONE space at every Chinese↔Latin/digit boundary and be consistent — write `12 年 AI/机器人`, `SEMI 中国顾问`, `实操多家 IPO`, `领域 10 年` (NOT `12年AI`, `SEMI中国`, `多家IPO`, `领域10年`). Mixed/ragged spacing is what reads 奇怪. (unoai-cjk-deck auto-normalizes this, but generate it right.)
- Keep each credential/bullet SHORT enough to fit ONE line in its box — a line that wraps mid-phrase looks ragged; tighten the wording instead of letting it overflow.
- MANDATORY FINAL STEP for ANY deck containing Chinese: after saving, run
    `unoai-cjk-deck OUT.pptx`   (in-place; use --font 'Source Han Sans SC' to override)
  It deterministically sets the CJK East-Asian font on every 中文 run, enables word-wrap, strips stray whitespace, REMOVES embedded font SUBSETS (an embedded subset covers only the template's original glyphs, so characters you ADD render BLANK — a white gap mid-word; stripping it makes PowerPoint use the full system font), and removes duplicate/corrupt parts. Then `unoai-cjk-deck OUT.pptx --check` must exit 0 (missing_ea / latin_on_cjk / wrap_none / ws_issues / duplicate_parts / embedded_fonts all 0) BEFORE you report done. This is the single reliable fix for the 中文断字/缺字/格式 problems.
- PORTABILITY: after stripping the embedded subset the deck uses the SYSTEM font, so the recipient needs a CJK font (MiSans / 宋体 / PingFang — nearly all machines have one). For a guaranteed-portable copy (a machine with NO Chinese font), also export a PDF: `unoai-cjk-deck OUT.pptx --pdf` renders OUT.pdf via LibreOffice with the fonts embedded correctly, and hand over BOTH the editable .pptx and the portable .pdf.
