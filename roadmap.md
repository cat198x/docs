# Roadmap: from verifier to reorganiser

**Status:** Updated 2026-06-14. The reorganiser is **built and the TOSEC reorg
is executed**. The critical path (`M0`–`M4`) and quality items (`Q1`–`Q7`) all
landed. The **arcade phase is paused mid-flight**, deliberately: every *code*
blocker is fixed and merged (see below), but the arcade *data* reorg cannot be
finished until a **merge-mode feature** exists and the **Time Capsule is stable
with headroom**. Nothing is lost — see "Arcade phase — paused" for the exact
state and resume path. Design detail for the layout engine lives in
[`reorganise-layout-engine.md`](reorganise-layout-engine.md); this doc is the
**order, dependencies, and acceptance** across all of it.

## Arcade phase — paused (2026-06-14)

Scanned and catalogued into `~/.cat198x`: MAME (ROMs/Extras/Multimedia + CHDs),
the **MAME 0.288 Software List** set (712 collections, 234,485 entries), Demul,
Raine, FinalBurn Neo. **CHD support landed** (`2bdfd7a`, `7ccf228`, `c6cdd9a`,
`b5cfa9c`): the DAT parser reads `<disk>` elements; the scanner identifies a
`.chd` by its **internal header SHA1** (CHD v5, offset 84, header only); the
planner stores CHDs **loose** at `<dest>/<game>/<name>.chd`.

### Code blockers — all fixed and merged (five PRs)

| PR | Fix |
|----|-----|
| #8 | **`Q7` planner performance.** `files(crc32,size)` index (CRC-only arcade DATs were full-scanning the files table per ROM); completeness resolved by set-membership + rarest-entry instead of an `O(games×containers×entries)` scan; a memory guard skips meta-aggregates like `all_non-zipped_content` (62M match-rows). FinalBurn Neo: never-finished → 38s; full arcade plan → 74s. |
| #9 | **CHD copy verification.** A CHD is catalogued by its internal header SHA1; `execute_copy` was verifying the file-byte hash, so every shared-CHD copy failed. Now verifies `.chd` via `read_chd_sha1`. |
| #10 | **`prune-empty`** command — clears empty source dirs a `--move` tidy leaves (only `fs::remove_dir`). |
| #11 | **`apply --prune-empty`** — self-cleans at end of run. |
| #12 | **`Q6` convergence (`catalogue-placements`) + dedup guard.** The library dest wasn't a registered source, so the apply-time sync dropped placements and every re-plan re-listed (and would re-transfer) them. `catalogue-placements` registers the library and backfills placements from a completed plan's op-log (no re-hashing); afterwards re-plans converge and future applies record their own placements. **Critical dedup guard:** with the library catalogued, the move-mode dedup cross-deleted placed files for merged-set parent/clone-shared ROMs (one DAT game → flagged not-shared → 4.6M deletes, 99.99% under `Library/ROMs`). The guard never deletes a file under a destination root. |

### Two non-code blockers stop the data reorg

1. **`merge_mode` is unimplemented in the planner.** `generator.rs` never reads
   `merge_mode`; `PlanOptions` has no merge field (it's used only by `status`).
   Placement is "whatever ROMs the DAT lists per game" + output format. The MAME
   DAT is *merged* (clones carry inherited ROMs with merge tags), so the planner
   places them per-clone — i.e. **non-merged**, regardless of config. Combined
   with `output_format = loose`, the apply *extracts* ROMs out of the compressed
   `ToSort` zips into loose files and *duplicates* inherited ROMs across every
   clone, while the source zips can't be deleted (they hold other games). That
   uncompress-and-duplicate is why ~175 GiB vanished placing sets that, as a
   `--move`, should have cost ≈0. **To finish arcade sanely, the planner needs a
   real merge-mode feature** — `split` (a clone's archive holds only its own
   ROMs; inherited ones live in the parent) is the chosen target. It must filter
   merge-tagged inherited ROMs out of clone placements. This is genuine MAME
   merge-semantics work, scoped deliberately, not bolted on.

2. **The Time Capsule drops its AFP connection** unpredictably (`os error 5`,
   then a stale-mount cascade of `EACCES`/`ENOENT` on every op). It killed the
   overnight apply (190,739 ✓ / 226,898 ✗) and the recovery apply (6,839 ✓ /
   220,362 ✗) — **not load or a bug; the mount died.** A resilient resume loop
   (`/tmp/cat198x-resume-loop.sh`: re-plan → apply, repeat; each round's
   successes are recorded so progress accumulates with no re-transfer) handles
   the drops, but the reorg can't complete until the TC is stable, and the
   non-merged layout won't fit anyway (TC ~113 GiB free).

### Exact state — safe, consistent, resumable

- **No data lost.** Verify-before-delete held through both failed applies; every
  failed op's source is intact in `ToSort/`. The placed library content is real
  and verified.
- **~197,650 ops placed and catalogued** (Demul + FinalBurn Neo + Raine + ~25%
  of MAME), as **loose non-merged** — the layout we're moving away from.
- **Catalogue:** `Library/ROMs` registered as source id 33; placements backfilled
  via `catalogue-placements --plan 9dcee803`. Re-plans converge (~197,650
  already-correct). Backups: `db.sqlite.bak-prechd-2026-06-13`,
  `db.sqlite.bak-preq6-2026-06-14`.
- **Dangerous plan quarantined:** `objects/plans/QUARANTINE-dangerous/ae13040e…`
  (the pre-guard 4.6M-library-delete plan). Do not apply.

### Resume path

1. Decide arcade layout with TC capacity in hand (likely **zip + split**).
2. Implement `split`/`merged` merge-mode in the planner (filter merge-tagged
   inherited ROMs per clone; route to parent). New `Q8`.
3. Reconsider the already-placed loose non-merged arcade content — re-plan to
   the new layout will repack/replace it; budget space for the transition.
4. Stabilise the TC (wired link / power-save / reboot), then drive the resilient
   resume loop to completion.

*(Historical `Q7` detail retained below for the record.)*

## Arcade phase — original `Q7` blocker (2026-06-13, resolved by PR #8)

The original hang: a full `--move` plan ran ~2h23m and stuck on
**`FinalBurn Neo - Arcade Games`** (100 min CPU). The upfront scan reported
**86,911 shared contents** and **103,802 containers sourcing multiple games**.
Root causes and fixes are in PR #8 above.

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

### Q5 — Parallelise `apply` (concurrent repacks)  *(✅ landed 2026-06-12, `df1d7db`)*

> **Done.** `apply` gained `-j`/`--jobs` (default 8, max 64): consecutive
> pending repacks batch and run on a bounded pool of worker threads, while the
> rollback journal, plan status, and catalogue sync stay on the calling thread
> in completion order (the rusqlite connection is not `Sync`). Any other
> pending operation flushes the batch first, so plan ordering between repacks
> and everything else is unchanged, as is dry-run. Workers run the same audited
> `execute_repack` — verification and move-mode delete-after-verify intact.
> The original rationale, kept as the design record:

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

### Q7 — Planner performance on merged arcade sets  *(✅ resolved 2026-06-14, PR #8)*

> **Done.** Fixed by `files(crc32,size)` index + set-membership/rarest-entry
> completeness + a meta-aggregate memory guard. FinalBurn Neo never-finished →
> 38s; full arcade plan → 74s. Original analysis kept below.

`plan` hangs on large *merged* arcade collections. A `--move` plan over the
arcade sets ran 2h23m and never got past `FinalBurn Neo - Arcade Games` (100 min
CPU, then AFP I/O wait). Root cause: merged arcade DATs share ROMs across
parent/clone wall-to-wall — the run reported **86,911 shared contents** and
**103,802 multi-game containers**. The archive branch in `generator.rs` resolves
each game's canonical container by scanning candidate containers and running
`is_complete` (expected entries × container entries) per game; with thousands of
games each sharing many containers this is effectively O(games × containers) and
does not finish. Small sets (`32x`, `Demul`, `Raine`) plan in seconds; the big
merged sets (FBNeo Arcade, MAME ROMs merged) do not.

**Fix direction:** precompute a sha1 → containers (and container → entries) index
once, so per-game completeness is a lookup, not a scan; and/or short-circuit the
"complete container already at dest" case before the candidate scan. Likely also
revisit `compute_shared_containers` cost. **Acceptance:** a full arcade `--move`
plan completes in minutes. **Size:** M–L. **Do before re-running the arcade plan.**

### Q8 — Merge-mode in the planner  *(BLOCKER for finishing arcade — surfaced 2026-06-14)*

The planner ignores `merge_mode` entirely (`generator.rs` never reads it;
`PlanOptions` has no merge field — it's used only by `status`). It places
whatever ROMs the DAT lists per game, so a *merged* MAME DAT (clones carry
inherited ROMs with merge tags) is placed **non-merged**: every clone gets its
own copy of the parent's shared ROMs. With `output_format = loose` this also
*uncompresses* (extract from `ToSort` zips → loose files) and can't free the
source zips (they hold other games) — together a massive storage blow-up
(~175 GiB consumed where a `--move` should cost ≈0; the TC won't hold it).

**Fix direction:** implement `split` (and `merged`) in the planner — thread
`merge_mode` into `PlanOptions`/`generate_plan_filtered`, and for `split` exclude
merge-tagged inherited ROMs from a clone's placement so a clone's archive holds
only its own ROMs (inherited ones live in the parent). Pair with
`output_format = zip`/`torrentzip` for arcade so the result is *smaller* than the
`ToSort` input and the sources can be deleted. **Acceptance:** a `--set MAME
--move` plan with `merge_mode = split` produces per-game archives with no
inherited-ROM duplication, fitting in available space. **Size:** L. **Blocks
re-running the arcade data reorg.**

## Dependency view

```
Q1 ─┐
     ├─> M0 ─> M1 ─> M2 ─> M3 ─> M4   ✅ executed (TOSEC reorg done)
Q2 ──────────────────┘ (folded in)
Q3   ✅  Q4 ✅  Q5 ✅ (parallelise apply — landed 2026-06-12)
CHD support ✅ (2026-06-13: <disk> parse + internal-SHA1 scan + loose layout)
Q6   ── deferred (WHDLoad layout + extract convergence)
Q7   ── BLOCKER: planner perf on merged arcade sets (fix before arcade plan)
arcade phase  ── Q7 ─> full plan ─> apply (ToSort/ ~359 GB + MAME 0.288 SL set)
```

## Original goal: delivered — what's next

The in-place tidy you asked for was `M4`, and it ran: the TOSEC/ISO/PIX/WHDLoad
sets are reorganised in place, ~417,500 operations applied and verified. `Q5`
(concurrent repacks) landed 2026-06-12, clearing the arcade phase's gate. The
forward path is:

1. **Arcade phase.** Reorganise `ToSort/` (~359 GB: MAME / FinalBurn Neo / Demul
   / Raine + WOS-Archive, plus the MAME 0.288 Software List ROMs copied to the
   volume 2026-06-12) into `Library/ROMs`. The reorg engine is proven; this is
   mostly input scale, and the repack volume now runs `-j 8`-wide.
2. **`Q6` — WHDLoad layout + convergence.** Tidy the deferred WHDLoad
   category-level layout and the non-converging extract re-list. Non-blocking.
