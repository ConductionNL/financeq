# ADR-002: Spec-branch sync gate and template-provenance check

## Status
Proposed

## Date
2026-07-07

## Context

financeq has **zero implementation** on every ref that was checked: `origin/main`,
`origin/development`, `chore/rename-mydash-to-launchpad` (current branch), and all 21
`origin/spec/*` branches contain no `appinfo/`, `lib/`, or `src/` directory (verified via
`git ls-tree -r --name-only` across every ref). The only checked-out content on `development`
is `README.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`, `img/`,
and a single architecture doc, `openspec/architecture/adr-001-information-architecture.md`.
`openspec/changes/` and `openspec/specs/` are both empty/absent on `development`.

Meanwhile 21 `spec/*` branches exist with fully-formed `openspec/changes/<name>/` content
(`proposal.md`, `tasks.md`, `specs.md`, `design.md`, `hydra.json`, `context-brief.md`) —
these are `opsx-ff` outputs that were never synced into `development` per the documented flow
(`opsx-new` → `opsx-ff` → `opsx-plan-to-issues` → `opsx-apply` → `opsx-verify` → `opsx-archive`).
ADR-001 (financeq, 2026-05-23) already assumes these 21 branches exist and assigns each a menu
placement, but it does not establish who is responsible for landing them or in what order.

Worse, inspecting the proposals themselves shows **7 of the 21** carry a proposal title
referencing a different Conduction app, `Shillinq`, instead of `financeq`:

| branch | proposal.md first heading |
|---|---|
| `spec/budget-planning-control` | `# Proposal: Budget Planning & Control — Shillinq` |
| `spec/treasury-cash-management` | `# Treasury & Cash Management — Shillinq` |
| `spec/tax-levy-management` | `# Proposal: Tax & Levy Management — Shillinq` |
| `spec/obligation-financial-administration` | `# Proposal: Obligation & Financial Administration — Shillinq` |
| `spec/financial-reporting-accountability` | `# Proposal: Financial Reporting & Accountability — Shillinq` |
| `spec/grant-subsidy-management` | `# Proposal: Grant & Subsidy Management — Shillinq` |
| `spec/cost-accounting-allocation` | `# Summary` under `# Proposal: Cost Accounting & Allocation — Shillinq` |

These 7 proposals also read as generic ERP/commercial demand-analysis output (a
`.specter-prompt.txt` file is present alongside them, and their bodies cite "Demand Score",
"Multi-location budget management", "Direct material procurement with BOM management" — none of
which are gemeente/BBV concepts). By contrast, `spec/pc-cyclus-workflow`,
`spec/berap-marap-bestuursrapportages`, `spec/iv3-aanlevering-cbs`, and
`spec/driver-based-forecasting` are correctly titled and use BBV/Iv3/CBS terminology specific to
financeq. This means the 7 mislabeled branches are stale generic drafts from a shared
feature-demand corpus (used to seed multiple gemeentelijke/ERP apps, including Shillinq) that
were branched for financeq but never rewritten for the gemeente P&C domain that ADR-001 defines.
If synced and implemented as-is, financeq would ship private-sector ERP requirements (BOM
procurement, multi-location retail budgeting) under a public-sector P&C menu.

There is currently no mechanical gate that would catch "this spec's own proposal names a
different app" before a change is synced into `development` and handed to `opsx-apply`.

## Decision

### 1. No `spec/*` branch may be synced into `development` until it passes a provenance check

Before any of the 21 branches (or their successors) is merged into `development`'s
`openspec/changes/`, the responsible engineer/skill MUST grep the proposal, spec, and design
files for the literal strings `Shillinq`, `shillinq`, or any other app id not equal to
`financeq`/`FinanceQ`, outside of an explicit cross-app dependency reference (e.g. "reads
shillinq's OR register" is fine; "Proposal: ... — Shillinq" as the proposal's own title is not).
A hit in the proposal's own title or summary blocks the sync until the content is rewritten for
financeq's actual domain (BBV/Iv3/CBS/gemeente P&C), not merely find-and-replaced.

### 2. Sync order follows ADR-001's IA, tiered specs sync with their parent

`{module}-other-t1..t4` branches MUST be synced together with (same PR as, or immediately after)
their parent module branch, never standalone — a tiered spec merged without its parent has no
menu to attach sub-pages to, and ADR-001's placement table becomes a dangling reference.

### 3. A repo is not "started" until `development` has at least one synced change and a scaffolded `appinfo/`

`main`/`development` holding only docs and zero `openspec/changes` + zero `appinfo/` is a valid
pre-implementation state, but it MUST NOT be treated as "in progress" by fleet audits — a spec
branch existing in isolation on `origin` does not count as financeq work having started for the
purposes of fleet status reporting (e.g. the manifest/readiness audits referenced in company
memory). The first sync + `appinfo/` scaffold is the definitional start of implementation.

## Consequences

### Positive
- Catches template/corpus leakage (a generic demand-analysis draft shipped under the wrong app's
  name) before it reaches `opsx-apply`, where it would be far more expensive to unwind.
- Gives the 21 orphaned branches an explicit, ordered path into `development` instead of leaving
  them to rot as unreferenced remote branches.
- Aligns fleet status reporting with actual implementation state instead of branch existence.

### Negative / cost
- The 7 mislabeled branches require a rewrite (not a mechanical rename) before they can be
  synced — proposal, `specs.md`, `tasks.md`, and `design.md` content must be re-derived from
  BBV/Iv3/CBS domain requirements, not just the heading.
- Adds a manual/skill-driven grep step to the sync path that does not exist for apps whose specs
  were authored directly against the target app.

### Neutral
- Does not change ADR-001's menu placement decisions; it only gates what may be synced into the
  structure ADR-001 already defines.

## References
- `openspec/architecture/adr-001-information-architecture.md` (financeq) — defines the six-menu
  IA and the tiered-spec placement table that the synced branches must land in.
- ADR-022, ADR-024 (hydra) — apps consume OR abstractions, app manifest.
- CLAUDE.md (apps-extra) — canonical opsx flow: `opsx-new` → `opsx-ff` → `opsx-plan-to-issues` →
  `opsx-apply` → `opsx-verify` → `opsx-archive`.
