---
kind: code
---

## Why

financeq is not an app yet — it is a scaffold repo with no installable Nextcloud app in it.
At HEAD the repository root contains only community files, `img/`, `.forgejo/` and
`openspec/architecture/adr-001-information-architecture.md`. There is **no `appinfo/info.xml`,
no `lib/`, no `src/`, no `package.json`, no `composer.json`** — nothing Nextcloud can install
(`README.md:2` says "Financial bookkeeping engine extension (specs pending)").

Meanwhile the release plumbing already assumes an installable app:

- `.forgejo/workflows/release-stable.yml:3-9` triggers the shared
  `Conduction/.github` stable-release pipeline (`app-name: financeq`) on every push to
  `main` — against a repo with no `appinfo/info.xml` to package or sign.
- `.forgejo/workflows/release-beta.yml:3-9` does the same for `beta`.
- `.forgejo/workflows/documentation.yml:14-17` publishes `source-folder: docs`, but no
  `docs/` directory exists at HEAD (it was removed on `development`; `main` never had one
  restored) — the docs pipeline publishes an empty site.

And the app's own architecture contract is already binding but unimplemented:

- `openspec/architecture/adr-001-information-architecture.md:47-61` fixes exactly six
  top-level menus (Begroting, Treasury, Subsidies, Belastingen, Rapportage, Beheer) that MUST
  be encoded in an ADR-024 app manifest — no manifest exists.
- `adr-001-information-architecture.md:139-145` requires persona-driven menu visibility
  (medewerkers see five menus, concerncontrollers six) — nothing encodes it.

Peer apps (pipelinq, procest, decidesk, softwarecatalog) all ship the ADR-024 manifest-first
shell via `@conduction/nextcloud-vue` (`CnAppRoot` + `src/manifest.json`), ADR-044
`buildManifest` fragments, and ADR-042 first-time-setup gating. financeq should be born
conforming instead of retrofitted later.

## What Changes

- **Create the installable Nextcloud app skeleton**: `appinfo/info.xml` (id `financeq`,
  EUPL-1.2, NC version range matching the fleet), `lib/AppInfo/Application.php`,
  navigation registered in PHP (the `info.xsd` nav-route gotcha), `composer.json`,
  `package.json` + webpack build wired to `@conduction/nextcloud-vue`.
- **Create the ADR-024 manifest** at `src/manifest.json` encoding exactly the six top-level
  menus of financeq ADR-001 §1, in the fixed order, with the tier-suffixed module sub-pages
  reserved as placeholder pages under their assigned parents (ADR-001 §2 placement table).
  No seventh menu is representable without an ADR amendment.
- **Adopt ADR-044 menu architecture**: shared `buildManifest` pipeline, `src/manifest.d/`
  fragment directory, Beheer entries flagged as settings-foldout material (ADR-001 §5/§6:
  taakvelden config, COA, connectors live under Beheer, never in the operational nav).
- **Adopt ADR-042 first-time-setup**: declare setup preconditions (OpenRegister available,
  financeq register imported, gemeente + boekjaar chosen) so the app renders a setup gate
  instead of empty pages on a fresh install. No tenant-keyed seeding before the gate passes.
- **Restore a minimal `docs/` tree** (landing page + placeholder per top-level menu) so
  `documentation.yml` publishes a real site again.
- **Persona-based menu visibility** (ADR-001 §7): Beheer hidden for reguliere medewerkers,
  visible for concerncontrollers — wired via group/manifest visibility, not a raw
  permission bitmask.
- NOT in scope: any module functionality (budgeting, treasury, tax…). Those arrive via the
  consolidated module specs (see sibling change `consolidate-spec-branches-to-canonical-home`)
  and the BBV tree schema (see sibling change `publish-bbv-programma-tree-schema`).

## Impact

- New: `appinfo/`, `lib/AppInfo/`, `src/` (manifest + entrypoint), `docs/`, build config.
- `.forgejo/workflows/*` become truthful: releases package a real app; docs publish content.
- Unblocks every one of the 21 module specs, which all presuppose an app shell to land in.
