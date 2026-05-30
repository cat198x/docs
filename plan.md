# Catalog198x — plan

## Bottom line

Catalog198x **is** the existing Romshelf rebuild, renamed into the 198x family
and evolved for its mission. Romshelf is already a mature, safety-first ROM
manager — we keep all of it. The work is three things, in order:

1. **Rebrand** `romshelf` → `catalog198x` (mechanical, one pass).
2. **Make the 198x `assets/` library a first-class managed collection** — the
   bridge that turns a generic ROM manager into *the family's* asset tool.
3. **The TOSEC-PIX job** — verify the scans, then *extract source material*:
   route the reference-worthy items (manuals, magazines) toward `reference/`
   and the cultural material toward the Vault.

This **resolves the open "how much rebuild" question** in the umbrella decision
record ([`catalog198x-asset-tooling.md`](../../decisions/catalog198x-asset-tooling.md)):
the answer is **rename + evolve, not rewrite**. The ~16k-LoC rebuild is good and
stays. Any from-scratch replacement of a subsystem must be justified here first.

## What we're starting from (the rescue asset)

Romshelf, at `~/Projects/ROMShelf/romshelf` — a single-crate Rust CLI (edition
2024, ~16k LoC, unpushed rebuild), with a clean architecture and, importantly, a
**safety model already in place**:

| Area | What's there |
|------|--------------|
| **CLI** (`src/cli/`, `src/main.rs`) | `init`, `dat`, `source`, `scan`, `status`, `stats`, `config`, `plan`, `apply`, `quarantine`, `torrent`, `doctor`, `export`, `completions`, `update` |
| **DAT** (`src/dat/`) | clrmamepro + XML parsers, streaming for large files |
| **Scanner** (`src/scanner/`) | multi-hash (CRC32, MD5, SHA1, SHA256), archive scanning (zip, 7z), ROM header detection |
| **DB** (`src/db/`) | SQLite (bundled) — dats, files, collections, quarantine, config; cached hashes |
| **Plan/apply** (`src/plan/`, `src/cli/apply.rs`) | generate a reorg plan, then apply it — with **dry-run**, **rollback**, and **continue-rollback** |
| **Safety** | unknowns → **quarantine**; disk-space check before apply; Ctrl-C handling |
| **Distribution** | `fetch` (DAT download over HTTP), `self_update` from GitHub releases, shell completions |

The `plan → apply` split with dry-run + rollback already embodies Catalog198x's
top bar: **never lose or misidentify a file.** We are not rebuilding that — we're
inheriting it.

## Phase 1 — Rebrand romshelf → catalog198x

A single, mechanical rename pass. Land it as the first commit(s) in
`catalog198x/catalog198x` after migrating the rebuild in.

- **Crate + binary**: `romshelf` → `catalog198x`. (Open question below: the
  binary name is long to type — decide on a short alias, e.g. `cat198`, before
  this lands, since renaming a published binary later is disruptive.)
- **Cargo.toml**: `name`, `description`, `repository` → `catalog198x/catalog198x`,
  `keywords`/`categories` tweaked (cataloguing, preservation, not just "mame").
- **Data dir**: `~/.romshelf` → `~/.catalog198x`; env vars `ROMSHELF_CONFIG` /
  `ROMSHELF_DATA_DIR` → `CATALOG198X_*`. **Migrate, don't strand**: on first run,
  detect a legacy `~/.romshelf` and offer to move/copy it (the DB schema is
  unchanged, so this is a directory move).
- **`self_update` target**: `romshelf/romshelf` releases → `catalog198x/catalog198x`.
- **User-facing strings**: command `about` text, log lines, the `init` banner,
  `doctor` output.
- **Tests**: the 1,932-LoC integration suite must stay green through the rename;
  it's the proof the rebrand changed names and nothing else.

Everything else — the architecture, the SQLite schema, the plan/apply/rollback
machinery — is untouched in this phase.

## Phase 2 — The 198x `assets/` library as a first-class collection

This is what makes it *Catalog198x* rather than a generically-renamed Romshelf.

- The umbrella `198x/assets/` library (ROMs, OS disks, test suites, TOSEC sets)
  becomes a **named Catalog198x collection** with a known destination layout, so
  "the assets library" is something Catalog198x verifies and maintains by name,
  not an ad-hoc directory.
- Define the **library-layout convention** (documented alongside this plan) that
  Emu198x expects when it reaches into `assets/` — so a verified collection lands
  where the emulator looks for it. This is the Catalog198x → Emu198x contract.
- Catalog198x **feeds** Emu198x; it does not emulate. The boundary stays clean.

Nothing here needs new core machinery — it's `source`/`config`/`plan` pointed at
the family's own library with a settled layout. The new part is *documenting the
layout as a contract* and treating the assets library as the canonical collection.

## Phase 3 — TOSEC-PIX: catalogue, verify, extract

The first real job, and the one in flight (the downloads on the Time Capsule).

1. **Verify** the TOSEC-PIX download against its TOSEC DAT. *To confirm against
   the code:* TOSEC ships clrmamepro-format DATs, which Romshelf's `dat` parser
   should already handle — but TOSEC-PIX entries are **scan archives** (manual,
   box, and media scans), not ROMs, so confirm the scanner hashes archive
   *members* the way the PIX DAT expects before trusting a green checkmark.
2. **Catalogue**: `status`/`stats` give the completeness picture — what's
   verified, duplicated, missing.
3. **Extract source material** — the novel, 198x-specific capability, and the
   reason TOSEC-PIX matters to the family. PIX is scanned manuals, magazines,
   box art, and adverts. Catalog198x should **route** verified items by kind:
   - **Manuals / magazines / datasheets** → staged for the `reference/`
     ingestion pipeline (then docling-extracted — this closes the loop with the
     reference-library work).
   - **Box art / adverts / cultural ephemera** → toward the Vault (Code198x),
     per the scope axes.

   This routing is the one genuinely new subsystem. It is *additive* — a new
   command (e.g. `extract` / `route`) over the existing verified catalogue — not
   a change to the core. Keep it dry-run-first like `apply`.

## Decisions to capture (in `catalog198x/decisions/`, once migrated)

- **Rebuild scope = rename + evolve** (ratifies the umbrella record's open
  question with this plan as the evidence).
- **Data-dir migration** policy (`~/.romshelf` → `~/.catalog198x`).
- **Binary name** (full `catalog198x` vs a short alias).
- **The `assets/` library-layout contract** with Emu198x.
- **Extraction routing rules** — which TOSEC-PIX categories go to `reference/`
  vs the Vault, and in what staged form.

## Out of scope / open questions

- **No rewrite of the core.** Scanner, DB schema, plan/apply, quarantine stay.
- **Binary name** — `catalog198x` is correct-but-verbose; decide an alias.
- **TOSEC-PIX DAT specifics** — verify archive-member hashing matches the PIX DAT
  before relying on verification (don't assume; test against a real PIX DAT).
- **Extraction depth** — Phase 3's routing could stop at "stage the files for a
  human/agent to ingest" rather than driving docling directly. Start shallow.

## Sequencing

Phase 1 is a prerequisite and is cheap (a rename pass + green tests). Phase 2 is
small (config + a documented layout). Phase 3 is where the real new work and the
live TOSEC-PIX value are — but it sits on Phases 1–2 and on the Time Capsule
being mounted. Do them in order; don't start Phase 3's `extract` command before
the assets-library collection (Phase 2) gives it a destination.
