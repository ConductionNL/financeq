---
kind: config
depends_on:
  - bootstrap-installable-app-scaffold   # app skeleton + register import plumbing must exist first
---

## Why

financeq ADR-001 §3 (`openspec/architecture/adr-001-information-architecture.md:85-98`)
makes the BBV-programma-tree the single source of truth for the whole ecosystem:

> "The tree lives in openregister under the canonical schema published by financeq and
> must not be duplicated in app-local state."

Three consumers are already contractually bound to it — planix ladders projects to
programma's/doelen (ADR-001:26-27), Rapportage joins it for raads-rapportages and
BERAP/MARAP (ADR-001:91-92), and launchpad rolls it up for bestuurders via runtime GraphQL
only (ADR-001:176-181). Yet **no schema, register definition, or seed exists anywhere in
the repo at HEAD** — the repository contains zero JSON configuration of any kind. The
canonical contract that sibling apps must code against is currently unwritable and
unreadable: any planix/launchpad work against the tree today would have to invent its own
shape, producing exactly the duplicated-tree rot mode ADR-001 §3 exists to prevent.

Per hydra ADR-022 (apps consume OR abstractions) and ADR-031 (schema-declarative business
logic), this is a config head: schemas + register in the shared OpenRegister data layer,
no bespoke financeq endpoint, consumers read via plain OR object/aggregation APIs.

## What Changes

- **Declare the financeq OpenRegister register** (slug `financeq`) with the BBV tree
  schemas, imported via a Repair step on install/upgrade (OR does not self-import register
  JSON — ADR-037/Repair-step pattern).
- **`programma` schema** — BBV-conform node: `code` (budgetcode, unique within boekjaar),
  `naam`, `type` enum (`programma` | `doel` | `activiteit`), `parent` (self-relation UUID,
  null for root programma's), `boekjaar`, `taakveld` (Iv3 taakveld reference), `actief`.
  Hierarchy is parent-linked, not path-duplicated, so a rename propagates to every consumer
  automatically.
- **`taakveld` schema** — the Iv3 taakvelden lookup (CBS-governed configuration data,
  managed under Beheer per ADR-001 §5): `code`, `omschrijving`, `hoofdtaakveld`.
- **Read contract for consumers**: planix, Rapportage surfaces, and launchpad read the tree
  through standard OR object queries/aggregations on `financeq`/`programma`. Only financeq
  writes it (Begroting > Programmabegroting). Write access is enforced via OR schema RBAC
  (publish = RBAC, not `@self.published`).
- **No custom endpoint, no controller, no local copy in any consumer** (ADR-022). No
  seeding of gemeente-specific programma's before the ADR-042 setup gate passes; only the
  CBS taakvelden lookup may ship as neutral reference data.

## Impact

- New: `lib/Settings/` (or config-head location per current fleet convention) register +
  schema JSON; Repair step registration in `appinfo/info.xml`.
- planix and launchpad get a stable, versioned read contract; the ADR-001 §3 single-source
  rule becomes enforceable instead of aspirational.
- Namespacing: schema slugs are globally unique in OR (lower(slug) collision gotcha) —
  slugs are `financeq_programma` / `financeq_taakveld` style if the fleet convention
  requires prefixing; verify against current OR fleet practice at implementation time.
