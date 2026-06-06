# Roadmap: from verifier to reorganiser

**Status:** Draft, 2026-06-06. Sequences the work surfaced while driving the live
TOSEC cataloguing job. Design detail for the layout engine lives in
[`reorganise-layout-engine.md`](reorganise-layout-engine.md); this doc is the
**order, dependencies, and acceptance** across all of it.

## Bottom line

Cat198x already does the *analysis* half of the mission (inventory, verify,
completeness). The gap is the *reorganise* half — a cohesive hierarchical tidy.
Five milestones (`M0`–`M4`) form the **critical path** to that, in strict order.
Four quality items (`Q1`–`Q4`) are smaller and mostly parallel; two of them
(`Q1`, `Q2`) are sequenced into the path because they de-risk it cheaply.

Recommended order: **`Q1` → `M0` → `M1` → `M2` (+`Q2`) → `M3` → `M4`**, with `Q3`
shippable anytime as a quick analysis win and `Q4` deferred.

Each milestone is one PR's worth of work: builds clean, tests at the right level,
lands on its own branch in the flagship `cat198x/cat198x` repo. Standard cadence —
one change per commit, no skipped tests.

## Progress (2026-06-06)

Landed on branch `feat/layout-engine`, in order, each its own tested commit:

- **`Q1` ✅** `dat add` is idempotent — re-adding an existing version is a
  reported skip, not a `UNIQUE` error.
- **`M1` ✅** recursive `dat add` records each collection's relative library
  path on its node (the canonical-`DatRoot` nesting drives it).
- **`M2` ✅** `build_dest_path` lays single-ROM games flat and multi-ROM games in
  a game folder (loose layout).
- **`M3` ✅** plan iterates *all* collections and resolves each destination from
  an explicit `dest_path`, else a library-wide `default_dest_path` + the node's
  hierarchy, else a *reported* skip (no more silent skip).
- **`Q2` ✅ (already existed)** the destination free-space guard is already in the
  executor (`executor.rs`, 10% margin) — roadmap item was stale; not redone.
- **CLI ✅** `config set-default <key> <value>` sets the file-level defaults
  (`dest_path`/`output_format`/`merge_mode`), so `default_dest_path` no longer
  needs hand-editing.
- **End-to-end ✅** one integration test composes the whole chain: recursive add
  → `set-default` → plan lands the ROM at `<default>/<hierarchy>/<rom>`.

The layout engine is functionally complete for the loose sets. Remaining:
**`M0`** (canonical DAT source — needs Steve's decision) and **`M4`** (live
rebuild + dry-run), both gated on the scan finishing and not yet started because
they touch the live catalogue. `Q3`/`Q4` remain optional/deferred. Archive output
formats (zip/TorrentZip) in `build_dest_path` are still a later step.

## Baseline (in flight, not a roadmap item)

The completing scan is running; once done, load the missing TOSEC-main + TOSEC-ISO
DATs and produce `status`/`stats`/`export`. That delivers the analysis goal and
gives every milestone below a complete catalogue to work against. Nothing in the
roadmap blocks it.

## Critical path

### M0 — Canonical DAT source for hierarchy  *(decision + setup)*

**Why first:** hierarchy injection (`M1`) needs a reliable source for each
collection's place in the tree. Spec open-question #1.

**Decision needed (Steve):** how to get a complete, hierarchy-bearing DAT set.
- **(A) Nested `DatRoot/`, path-relative** *(recommended to ship first).* Complete
  the nested `DatRoot/` tree, register from it, derive hierarchy from each DAT's
  path relative to the set root. Simplest tool logic; tracks TOSEC's own layout.
  Cost: completing the partial nesting (1,103 vs 1,332 PIX) needs a one-time sort
  of the flat pack into folders.
- **(B) Name-parse in `dat add`.** Parse TOSEC's `Manufacturer System - Category -
  Title` names into nodes; works on the flat pack with no sort. Cost: fragile
  edge cases (e.g. "Nintendo Famicom & Entertainment System"); parsing logic
  lives in the tool permanently.
- **(Hybrid, end state)** path-relative when nested, name-parse fallback when flat.

**Drift-trigger note:** a *one-off* sort of DAT files into folders is file
organisation, not a verify/dedup script, so it does not hit the "no ad-hoc TOSEC
scripts" rule. But if it grows logic it must become a `cat198x` subcommand, not a
standalone script.

**Acceptance:** a single recursive `dat add` of the chosen source yields a
complete, correctly-nested set ready for `M1`. **Size:** S–M.

### M1 — Hierarchy injection at recursive `dat add`  *(spec §A)*

Build the `root → manufacturer → system → category → dat` node chain during
`dat add --recursive`, reusing nodes by `(version_id, path)`. Single-file
`dat add` keeps flat behaviour.

**Depends on:** `M0`. **Acceptance:** node tree mirrors the on-disk folders;
`SELECT DISTINCT node_type` returns more than `dat`. **Size:** M.

### M2 — Hierarchical `build_dest_path`  *(spec §B, + `Q2`)*

Compose `<base>/<hierarchy>/<title>/<rom_name>`; omit the redundant `<title>`
level for single-ROM loose games. Respect `output_format`/`merge_mode`; ship
`loose` first (covers TOSEC/ISO/PIX). Fold in `Q2` (free-space guard) here.

**Depends on:** `M1`. **Acceptance:** dry-run destinations are byte-identical to
real on-disk canonical paths; already-correct files produce zero operations.
**Size:** M.

### M3 — Base / per-set destination + fix silent-skip  *(spec §C)*

Add a per-set `base_dest` (and/or global default) so a whole set inherits a
destination derived as `base_dest + hierarchy`. Resolution order:
per-collection `dest_path` → per-set `base_dest` → global default → **skip with a
reported count** (replacing today's silent skip at `generator.rs:85`).

**Depends on:** `M2`. **Acceptance:** one `config set <set> base_dest …` covers
every collection in the set; the plan summary names any skipped. **Size:** S–M.

### M4 — Drive the in-place tidy to dry-run  *(the payoff)*

Rebuild registrations from the canonical source (remove + add — cheap, leaves the
scanned inventory untouched), set each set's `base_dest` to its existing root,
`plan`, then `apply --dry-run` for review. The actual `apply` is a separate,
explicitly-approved step. "Tidy in place" needs no new mode — it is `base_dest =
set root`.

**Depends on:** `M1`–`M3` (and `Q1` for a clean rebuild). **Acceptance:** a
reviewed dry-run plan for TOSEC/ISO/PIX/WHDLoad whose operations are only the
genuine renames/moves/dedups. **Size:** S orchestration + the apply.

## Quality items

### Q1 — Idempotent `dat add`  *(do early)*

Re-adding an existing version currently **errors** (`UNIQUE constraint failed`)
instead of skipping. Make it a reported no-op, and distinguish "already present"
from real parse failures in the recursive summary. Cheap, and it de-risks every
bulk add/rebuild (`M0`, `M4`). **Size:** S. **Do before `M0`/`M4`.**

### Q2 — Free-space guard  *(fold into `M2`/`M3`)*

`plan`/`apply` pre-check destination free space against required bytes and
warn/refuse. The set DATs describe ~2.6 TB against ~433 GB free, so this is a real
guard, not theoretical. **Size:** S.

### Q3 — Grouped / rollup stats  *(quick analysis win, anytime)*

Aggregate `status`/`stats` by set and by system (roll up "all TOSEC-PIX",
"all Sinclair ZX Spectrum - *") instead of 1,400 flat rows. Independent of the
critical path; useful for the analysis deliverable now. **Size:** S–M.

### Q4 — `dat relink`  *(deferred)*

Repoint moved DAT files without remove + add, clearing stale registrations (the
25 missing FinalBurn Neo). General hygiene; `M4`'s rebuild sidesteps the immediate
need, so defer unless wanted sooner. **Size:** M.

## Dependency view

```
Q1 ─┐
     ├─> M0 ─> M1 ─> M2 ─> M3 ─> M4   (critical path to reorg)
Q2 ──────────────────┘ (folded in)
Q3   (independent — analysis quality, anytime)
Q4   (deferred — hygiene)
```

## What unblocks the user's original goal

The in-place tidy you asked for is `M4`, and it cannot start before `M1`–`M3`.
Everything before that is either already working (analysis) or prerequisite
plumbing. The fastest honest route to a reviewable reorg dry-run is the critical
path above, with `Q1` done first so the catalogue rebuild is clean.
