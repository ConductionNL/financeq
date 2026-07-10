---
kind: mixed
depends_on: []
---

## Why

financeq has **no implementation on any branch**. `git ls-tree -r --name-only` against
`origin/main`, `origin/development`, and the current `chore/rename-mydash-to-launchpad` branch
shows no `appinfo/`, `lib/`, or `src/` directory anywhere; `development`'s `openspec/` holds
only `openspec/architecture/adr-001-information-architecture.md` — `openspec/changes/` and
`openspec/specs/` do not exist. Meanwhile 21 `origin/spec/*` branches carry fully-formed
`opsx-ff` output (`proposal.md`, `tasks.md`, `specs.md`, `design.md`, `hydra.json`,
`context-brief.md`) that was never synced into `development`, so the canonical opsx flow
(`opsx-new` → `opsx-ff` → `opsx-plan-to-issues` → `opsx-apply` → `opsx-verify` → `opsx-archive`,
per CLAUDE.md) stalled after `opsx-ff` and never reached `opsx-plan-to-issues`/`opsx-apply`.
ADR-001 (financeq) already assumes these 21 branches exist and assigns each a menu placement
under its six-menu IA, but nothing has landed them.

Inspecting the 21 branches' `proposal.md` first-heading shows **7 are mislabeled**: their own
proposal title names a different Conduction app, `Shillinq`, not `financeq`
(`spec/budget-planning-control`, `spec/treasury-cash-management`, `spec/tax-levy-management`,
`spec/obligation-financial-administration`, `spec/financial-reporting-accountability`,
`spec/grant-subsidy-management`, `spec/cost-accounting-allocation` — confirmed via
`git show origin/<branch>:openspec/changes/<name>/proposal.md | head`). Their bodies read as
generic private-sector ERP demand-analysis output ("Multi-location budget management", "Direct
material procurement with BOM management", "Demand Score: 12,110") rather than the gemeente
BBV/Iv3/CBS domain ADR-001 defines financeq around. `spec/pc-cyclus-workflow`,
`spec/berap-marap-bestuursrapportages`, `spec/iv3-aanlevering-cbs`, and
`spec/driver-based-forecasting` are, by contrast, correctly titled and use BBV/Iv3/CBS
terminology. See new ADR-002 (`openspec/architecture/adr-002-spec-branch-sync-and-provenance-gate.md`)
for the full provenance-gate rule this change implements.

Without this change, financeq cannot proceed to `opsx-apply` on any of its 21 planned specs
(nothing is in `development` to apply against), and if the 21 branches were synced as-is, 7 of
11 root modules would ship private-sector ERP requirements under a gemeente P&C menu.

## What Changes

- **Sync the 14 correctly-titled spec branches' `openspec/changes/<name>/` content into
  `development`**, in ADR-001 parent-before-tier order: `pc-cyclus-workflow`,
  `berap-marap-bestuursrapportages`, `iv3-aanlevering-cbs`, `driver-based-forecasting`,
  `budget-planning-control` + its three `-other-t1..t3` tiers, `treasury-cash-management` + its
  four `-other-t1..t4` tiers, `tax-levy-management` + its three `-other-t1..t3` tiers.
  (Note: `budget-planning-control` and `treasury-cash-management` and `tax-levy-management`
  parent branches are themselves in the mislabeled-7 list below and must be rewritten before
  their tiers can be usefully synced — tiers cannot attach to a parent menu structure that does
  not yet reflect the financeq domain.)
- **BREAKING for planning purposes: flag the 7 mislabeled branches as blocked, not ready-to-sync.**
  `budget-planning-control` (+ 3 tiers), `treasury-cash-management` (+ 4 tiers),
  `tax-levy-management` (+ 3 tiers), `obligation-financial-administration`,
  `financial-reporting-accountability`, `grant-subsidy-management`,
  `cost-accounting-allocation` — 7 root branches plus their 10 dependent tier branches (17 of the
  21 total) — must have their `proposal.md`, `specs.md`, `tasks.md`, and `design.md` rewritten
  against financeq's actual BBV/gemeente domain (see ADR-001's per-module scope) before they can
  be synced. This change does the rewrite for the 7 root modules; the 10 tier branches are
  out of scope here and get their own follow-up changes once their parent's domain content is
  fixed (a tier spec inherits its parent's corrected scope).
- **Scaffold the minimal `appinfo/` per ADR-024 (app manifest)** so `development` has something
  for `opsx-apply` to target: `appinfo/info.xml` declaring the six ADR-001 top-level menus
  (Begroting, Treasury, Subsidies, Belastingen, Rapportage, Beheer) as navigation entries, and an
  empty `src/manifest.json` skeleton with those six pages and no widgets yet (widgets/pages are
  added by each individual synced spec's own `opsx-apply` run, not by this change).
- **Add the provenance grep as a documented pre-sync step** (this change's tasks.md) so future
  spec-branch syncs for financeq (and any other app whose specs were seeded from a shared
  demand-analysis corpus) re-run the same check before merging.

## Impact

- `openspec/changes/pc-cyclus-workflow/`, `openspec/changes/berap-marap-bestuursrapportages/`,
  `openspec/changes/iv3-aanlevering-cbs/`, `openspec/changes/driver-based-forecasting/` — new,
  copied from their respective `spec/*` branches unchanged.
- `openspec/changes/budget-planning-control/`, `openspec/changes/treasury-cash-management/`,
  `openspec/changes/tax-levy-management/`, `openspec/changes/obligation-financial-administration/`,
  `openspec/changes/financial-reporting-accountability/`, `openspec/changes/grant-subsidy-management/`,
  `openspec/changes/cost-accounting-allocation/` — new, content rewritten for financeq's BBV
  domain (not copied verbatim).
- `appinfo/info.xml` — new, minimal manifest with the six ADR-001 menus.
- `src/manifest.json` — new, six-page skeleton, no widgets.
- No existing financeq code is touched or deleted — there is none.
