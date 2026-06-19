# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is **not an application** — it is a distributable **Claude Agent Skill** (`gorden-ppt-skill`) that teaches an AI to generate/edit Chinese-language PowerPoint decks from 19 built-in `.pptx` templates (or a user-supplied template) by replacing only text, never layout. The repo ships as the skill payload itself.

Two distinct audiences read files here, and the distinction matters when editing:
- **`SKILL.md`** is the *runtime* instruction set — read by an AI *using* the skill to make a deck for an end user. Its rules (the "编辑铁律" / editing laws) are behavioral contracts; changing them changes how every downstream AI behaves.
- **`CLAUDE.md`** (this file) is for *developing/maintaining* the repo.

If you edit skill behavior, the change usually lives in `SKILL.md` + the `references/` docs, not in code.

## Commands

```bash
# Build a deck: select slides + replace text from a template. Do NOT pass --strict
# (overflow detection is advisory by design — see "Editing laws" below).
python3 scripts/build_pptx.py <template.pptx> <edits.json> <out.pptx> --detail <detail.json>
python3 scripts/build_pptx.py ... --dry-run     # print planned edits, write nothing

# Render any pptx to one PNG per slide (preview / self-check). Needs LibreOffice + poppler.
python3 scripts/render_slides.py <in.pptx> <out_dir> --dpi 144

# Recompute capacity fields (chars_per_line/max_lines/max_chars + type_scale) for a template.
# Only needed when ADDING or changing a template — built-in templates are precomputed.
python3 scripts/compute_capacity.py <detail.json> <template.pptx> --in-place

# Dependency check
python3 -c "import pptx; print(pptx.__version__)"   # python-pptx 1.0+
soffice --version    # LibreOffice (render only)
which pdftoppm       # poppler (render only)
```

There is **no test suite, linter, or package manifest**. Verification is manual: build a deck, render it, eyeball the PNGs.

## The two-file data contract (core architecture)

Everything flows through two JSON shapes, fully specified in `references/pptx-edit-schema.md`:

- **`detail.json`** (one per template, authored by the template maintainer): describes every slide's `role`, every editable **slot** with a stable `slot_id`, a physical `address` (`{shape_id, paragraph, run}`), `current_text`, capacity fields, and a per-slot `level` keyed to the template-root `type_scale` (font-size tiers). `page_roles` indexes which slides are cover/agenda/section_divider/content/ending.
- **`edits.json`** (authored at runtime by the editing AI): `selected_slides` (1-based, **unselected slides get deleted**) + `edits[]` that each target a slot by `slot_id` (needs `--detail`) or explicit `address`, with `new_text`.

`build_pptx.py` resolves `slot_id`→`address` via `detail.json`, mutates runs **in template-index order before pruning**, then keeps only `selected_slides`. Run-level formatting is preserved: replacing a whole paragraph writes into run 0 and clears the rest, keeping run 0's font/size/color. `find_shape` recurses into groups.

When a semantic value spans multiple paragraphs/shapes, it must be split into multiple `slot_id`s — cross-paragraph/cross-shape slots are unsupported.

## Three operating modes (runtime, defined in SKILL.md)

- **Mode A** — pick from built-in templates (default path). Workflow in `references/workflow.md#mode-a`.
- **Mode B** — user brings their own `.pptx`; probe it live with python-pptx + render, write edits by explicit `address`. Never modify the user's original file. See `references/custom-template-workflow.md`.
- **Mode C** — fully original design, generated directly with python-pptx. See `references/original-design-guide.md`.

## Editing laws that constrain the code

These are load-bearing — code changes that violate them are bugs, not improvements:

- **Overflow detection is advisory and never blocks.** `max_chars`/`chars_per_line`/`max_lines` are soft capacity hints (with ~20% tolerance). `check_overflow` only prints notes; even `--strict` must never reject a save over length. The historical worst-case the skill actively fights is **ellipsis truncation** (`...`/`…`/`等等`) added to fit a box — `build_pptx.py` flags it separately. Default builds run without `--strict`.
- **Only text changes.** Position, size, color, font, size, line-spacing are never touched.
- **`editable: false` slots** (decorative `01`/`02`/`%` etc.) are left alone.
- **Same `level` → same font size.** Length is controlled by rephrasing, not by shrinking one slot's font.
- **Use only the roles a template has** — empty `page_roles.cover`/`ending`/`agenda`/`section_divider` means omit that page, don't fabricate one.

Capacity is measured in CJK-equivalent **visual-width units** (CJK glyph = 1.0, latin ≈ 0.5, space ≈ 0.35). `_visual_width` in `build_pptx.py` and `vw_of` in `compute_capacity.py` must stay in sync.

## Versioning & self-update mechanism

The skill updates itself like software:

- `VERSION` (single line, e.g. `1.0.17`), `CHANGELOG.md` (human), `updates.json` (machine: `latest_version` + per-version `added/modified/removed` deltas + `update_source`), `manifest.json` (sha256 of every tracked file).
- `apply_update.py` (the **first tool call every session per SKILL.md**) clones `update_source` (`git+https://github.com/GordenSun/GordenPPTSkill.git#main`), computes the delta from local `VERSION`, downloads only changed files, verifies sha256, rewrites `VERSION`+`updates.json`. `check_update.py` previews without applying.
- Templates are **plain git blobs, not LFS** (each `.pptx` kept <14MB, fonts stripped) — see `.gitattributes`. The LFS-selective-pull path in `apply_update.py` is legacy fallback.

**When making a release**, all of these must move together for the same version: bump `VERSION`, add a `history` entry to `updates.json` listing exactly the changed files, append to `CHANGELOG.md`, and run `python3 scripts/build_manifest.py` to refresh `manifest.json`. The git commit convention is `vX.Y.Z: <summary>`.

## Templates

Each lives in `templates/<slug>/` with `template.pptx`, `detail.json`, `intro.md` (condensed brief), `preview.png` (2×2 montage of 4 slides). `templates/INDEX.md` is the selection table (slug, name, page count, primary color, one-line style) the runtime AI reads first. To add a template: create the folder, author `detail.json` (or scaffold then run `compute_capacity.py`), then register it in `INDEX.md`.

## Licensing constraint

Templates are **non-commercial, study/research only** — third-party designer assets, no redistribution rights. This notice appears in README, SKILL.md, and INDEX.md; keep it intact.
