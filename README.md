# Catalog198x documentation

Documentation for [Catalog198x](https://github.com/catalog198x), the binary-asset
cataloguing tool.

**Start here:** [`plan.md`](plan.md) — the rescue-and-evolve plan for turning the
Romshelf rebuild into Catalog198x (rebrand → `assets/` integration → the
TOSEC-PIX catalogue/verify/extract job).

## Scope

- The **verification model** — how files are hashed and matched against reference
  DATs (No-Intro, TOSEC, Redump, and friends).
- **DAT handling** — the formats understood, and how DAT versions are tracked and
  upgraded.
- **Library-layout conventions** — how a verified collection is organised on disk,
  and how that feeds the 198x `assets/` library that Emu198x consumes.

## Not here

- The binary assets themselves — those live in the umbrella `assets/` library.
- Hardware facts — the umbrella `reference/` primary library and syntheses.

## Conventions

Plain Markdown for now. A generated docs site can follow once there's enough to
warrant it; nothing here assumes one.

Empty for now; structure follows as the rescued Romshelf rebuild lands and its
commands settle.
