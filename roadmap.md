# Roadmap: from verifier to reorganiser

**Status:** Updated 2026-06-09. The reorganiser is **built and the TOSEC reorg
is executed** — not just planned. The critical path (`M0`–`M4`) and the quality
items (`Q1`–`Q4`) all landed, and ~417,500 operations ran live against the AFP
volume to assemble `Library/ROMs/{TOSEC,TOSEC-ISO,TOSEC-PIX,WHDLoad}`. What
remains is the **arcade phase** (the 359 GB `ToSort/` input) plus two polish
items surfaced by the live run: `Q5` (parallelise `apply`, recommended before
arcade) and `Q6` (WHDLoad layout/convergence, deferred). Design detail for the
layout engine lives in [`reorganise-layout-engine.md`](reorganise-layout-engine.md);
this doc is the **order, dependencies, and acceptance** across all of it.

## Bottom line

Both halves of the mission now work: Cat198x does the *analysis* (inventory,
verify, completeness) **and** the *reorganise* — a cohesive hierarchical tidy,
proven by executing it across 482,182 catalogued files. The original goal (an
in-place tidy of the TOSEC sets) is **delivered**. The remaining roadmap is
forward work: the arcade romsets, and the two efficiency/layout items the live
reorg exposed.

Each milestone was one PR's worth of work: builds clean, tests at the right
level, lands on its own branch in the flagship `cat198x/cat198x` repo. Standard
cadence — one change per commit, no skipped tests.

## Reorg executed (2026-06-07 → 09)

The plan was not just dry-run — it was **applied**. ~417,500 operations ran live
against the AFP-mounted Time Capsule (`/Volumes/Data`) to build
`Library/ROMs/{TOSEC,TOSEC-ISO,TOSEC-PIX,WHDLoad}` from 482,182 catalogued files,
in three resumable passes:

- **Pass 1 — 302,214 ✓.** Renames, relocates, loose moves, and dedups (the
  metadata-cheap operations, ~25–40 ops/s over AFP).
- **Pass 2 — 112,250 ✓, 81 failed.** Repacks (read-entry + recompress + write +
  verify, ~3 ops/s — latency-bound, ~10h for ~11 GB). The 81 failures were
  duplicate-entry-name collisions.
- **Pass 2b — 3,062 ✓, 1 failed.** Retried the 81 (all built once repack
  de-duplicated source entry names) plus follow-on work. The single straggler
  was a CP437-encoded archive entry name (`å`), now fixed.

**Lesson — latency dominates, not bytes.** Over a network mount the per-operation
round trip is the cost; the whole effort was tuned around that (renames over
copies, trust the catalogue, defer/parallelise repacks). Captured in the KB:
*Large ROM reorg over a network mount*.

Fixes that landed **mid-run**, each its own tested commit, as the live apply
exposed them:

- **`1c93afa`** apply keeps the catalogue in step per-op — so a re-plan converges
  with no re-scan, and a retry resumes instead of re-listing.
- **`ce9276c`** resumable partial plans + `--skip-repack` (defer the slow pass).
- **`5d8c6a1`** repack loose files into archives instead of renaming them to
  `.zip`; **`0785e04`** move-mode repack deletes its loose sources, reversibly.
- **`1f75a2b`** same-filesystem loose move renames without re-hashing — the
  single biggest AFP win (no full read-back of every ROM).
- **`7407d03`** quarantine store configurable, default on-volume (a cross-device
  byte-copy had been filling the local disk).
- **`5844321`** delete exact-content duplicates instead of quarantining them.
- **`7204e2a`** `truncate_path` no longer panics on multi-byte UTF-8 paths
  (crashed pass 1 at op 4,721 on `Año`).
- **`872749a`** content shared across entries is copied to each game, never
  consumed; **`0a21ad1`** a container sourcing multiple games is repacked
  per-game, never relocated/deleted whole (stranding siblings).
- **`9c6f678`** repack collapses duplicate entry names instead of aborting
  (cleared the 80 pass-2 failures).
- **`d4a7f11`** ZIP entry lookup finds CP437-encoded (non-UTF8-flag) names by
  decoded `name()` rather than the `zip` crate's inconsistent `by_name` map
  (cleared the last straggler — the `å` `.lha`).

`ToSort/` still holds ~359 GB — the arcade-phase input (MAME / FinalBurn Neo /
Demul / Raine + WOS-Archive + misc), the next major reorg.

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

- **`Q3` ✅** `stats --group` rolls collections up by leading name segment.
- **`Q4` ✅** `dat relink <dir>` re-points registrations whose DAT file moved.

The layout engine is functionally complete for the loose sets, and all the
quality items are in. **`M0` is now decided** — nested `DatRoot`, path-relative,
added from the `DatRoot` parent with `default_dest_path = /Volumes/Data` — see
[`m0-canonical-dat-source.md`](m0-canonical-dat-source.md) for the decision and
the post-scan procedure. Remaining is the operational sequence **`M0` verify/
complete + `M4` rebuild + dry-run**, gated on the scan finishing (it saturates
the AFP mount).

- **Archive output ✅** zip/TorrentZip collections now plan one archive per game
  (canonical entry names, content-verified already-correct check, converges on
  re-run). Loose remains the default for the TOSEC sets.

## Usability batch (2026-06-06, branch `feat/usability-batch`)

Ten follow-ups, each its own tested commit:

- **U1** `config get-default` + defaults shown in `config list`.
- **U2** `doctor` points at `dat relink` when DAT files are missing.
- **U3** `plan` writes the skipped (no-destination) collections to a file.
- **U4** `stats --group-by system|set` (was the boolean `--group`).
- **U5** `plan` prints a per-set breakdown of pending operations.
- **U6** archive already-correct check also verifies the container format
  (a content-correct plain ZIP no longer satisfies a TorrentZIP target). *Noted:
  cat198x's TorrentZIP still omits the canonical `TORRENTZIPPED` EOCD comment.*
- **U7** `plan` trusts the catalogue instead of re-hashing each destination file
  over the network — the in-place plan no longer re-reads the whole library.
- **U8** `plan --move` for a true in-place tidy (relocate, not duplicate), plus
  `unknowns` — a read-only report of files matched by no active DAT. *Auto-
  quarantine of unknowns deliberately deferred: it would sweep DAT-less folders.*
- **U9** `dat sort` nests a flat DAT pack by collection name (`" - "`-segmented).
- **U10** `7z` output format (native via sevenz-rust2).

## Baseline (done — not a roadmap item)

The completing scan finished and the full TOSEC-main + TOSEC-ISO + PIX + WHDLoad
catalogue (482,182 files) was the inventory every milestone ran against. The
analysis deliverable (`status`/`stats`/`export`) works on that complete catalogue.

## Critical path  *(all executed — `M0`–`M4` ✅)*

> **Done.** Every milestone below shipped and then *ran* against the live
> library. `M0` was decided (A: nested `DatRoot`, path-relative), `M1`–`M3` built
> the hierarchy/destination engine, and `M4` drove it past dry-run into the real
> apply documented in *Reorg executed* above. The descriptions are kept as the
> design record; the acceptance criteria were met in production.

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

### M4 — Drive the in-place tidy (dry-run → executed)  *(the payoff)*

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

### Q5 — Parallelise `apply` (concurrent repacks)  *(before the arcade phase)*

`apply` runs operations strictly one at a time. Renames/relocates/deletes are
metadata-cheap, but a **repack** is read-entry + recompress + write-zip + verify,
each a round trip over the AFP mount — latency-bound at ~3 ops/s. The live TOSEC
reorg's pass 2 was ~109k repacks ≈ **~10h** for only ~11 GB of data: the cost is
per-file round trips, not bytes. Repacks are independent, so applying them
concurrently (≈8–16 in flight) overlaps the network latency and should cut this
to well under an hour.

**Why now:** matters most before the **arcade phase** — MAME/FinalBurn romsets
are almost entirely multi-ROM containers, so they generate far more repacks than
TOSEC, where this would otherwise dominate the wall-clock.

**Correctness constraints:** the rollback log and the apply-time catalogue-sync
must stay correct under concurrency (both are per-op and independent, so this is
a bounded-worker fan-out, not a redesign); keep the disk-space guard and the
move-mode source deletion intact. Surfaced driving the live reorg. **Size:** M.

### Q6 — WHDLoad layout + extract convergence  *(deferred — surfaced in live reorg)*

Two linked WHDLoad issues found retrying the reorg's repack stragglers:

1. **Dest drops the category level.** WHDLoad collections are registered from the
   category subdirs (`WHDLoad - Applications`, `…- Cracktros`, `…- Demos`,
   `…- Games-*`), but the planned dest lands the extracted `.lha` *flat at the
   WHDLoad root* (`WHDLoad/<rom>.lha`) — the per-category level isn't applied, so
   all categories pile into one directory (unlike TOSEC/PIX, which keep their
   deep hierarchy). Decide the intended WHDLoad layout (likely
   `WHDLoad/<category>/<rom>.lha`) and fix the hierarchy resolution for it.
2. **Extracts don't converge.** Those `.lha` are extracted from their `.zip`
   wrappers to the WHDLoad root, but the root isn't a registered source (only the
   category subdirs are), so the apply-time catalogue-sync can't record the loose
   copy at its dest — every re-plan re-lists the ~2,983 extract-moves. Non-
   destructive (idempotent re-extraction, source kept) but never clears. Fixing
   (1) likely fixes (2); otherwise register the dest root (or the whole
   `Library/ROMs`) as a source so placed files catalogue and converge. **Size:** M.

## Dependency view

```
Q1 ─┐
     ├─> M0 ─> M1 ─> M2 ─> M3 ─> M4   ✅ executed (TOSEC reorg done)
Q2 ──────────────────┘ (folded in)
Q3   ✅  Q4 ✅
Q5   ── recommended before the arcade phase (parallelise apply)
Q6   ── deferred (WHDLoad layout + extract convergence)
arcade phase  ── next major reorg (ToSort/ ~359 GB), benefits from Q5
```

## Original goal: delivered — what's next

The in-place tidy you asked for was `M4`, and it ran: the TOSEC/ISO/PIX/WHDLoad
sets are reorganised in place, ~417,500 operations applied and verified. The
forward path is:

1. **`Q5` — parallelise `apply`.** The arcade romsets are overwhelmingly
   multi-ROM containers, so they generate far more repacks than TOSEC. At the
   current ~3 repacks/s that would dominate wall-clock; bounded-concurrent
   repacks should cut it to well under an hour. Recommended **before** the arcade
   phase.
2. **Arcade phase.** Reorganise `ToSort/` (~359 GB: MAME / FinalBurn Neo / Demul
   / Raine + WOS-Archive) into `Library/ROMs`. The reorg engine is proven; this
   is mostly input scale and the repack volume `Q5` addresses.
3. **`Q6` — WHDLoad layout + convergence.** Tidy the deferred WHDLoad
   category-level layout and the non-converging extract re-list. Non-blocking.
