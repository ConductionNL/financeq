# Spec Branch Lifecycle — financeq

**Spec refs**: ADR-001 (financeq — information architecture), ADR-002 (financeq — spec-branch
sync and provenance gate), ADR-024 (hydra — app manifest)

## ADDED Requirements

### Requirement: Spec branches MUST pass the provenance gate before sync

A `spec/*` branch MUST NOT be synced into `development`'s `openspec/changes/` unless its
`proposal.md`, `specs.md`, `tasks.md`, and `design.md` files contain no reference to a
Conduction app id other than `financeq` in their own title, summary, or requirement bodies
(cross-app dependency references, e.g. "reads shillinq's OR register", are permitted; the
branch's own proposal being titled for a different app is not). A branch that fails this check
MUST be rewritten against financeq's actual gemeente P&C/BBV domain before it may be synced.

#### Scenario: Branch titled for a different app is blocked

- GIVEN a `spec/*` branch whose `proposal.md` first heading reads
  `# Proposal: <Module> — Shillinq`
- WHEN a sync into `development` is attempted
- THEN the sync MUST be blocked
- AND the branch MUST be rewritten against financeq's BBV/gemeente domain before re-attempting

#### Scenario: Branch correctly scoped to financeq passes

- GIVEN a `spec/*` branch whose proposal, specs, tasks, and design content are scoped to
  financeq's BBV/Iv3/CBS domain (e.g. `pc-cyclus-workflow`, `iv3-aanlevering-cbs`)
- WHEN a sync into `development` is attempted
- THEN the sync MUST proceed

### Requirement: Tiered spec branches MUST sync with a corrected parent, never standalone

A `{module}-other-t1..t4` branch MUST NOT be synced into `development` before its parent module
branch has passed the provenance gate (Requirement above) and been synced. If the parent module
required a domain rewrite, the tiered branch MUST be re-derived from the corrected parent scope,
not synced against the stale parent content.

#### Scenario: Tier branch blocked ahead of its unrewritten parent

- GIVEN `budget-planning-control` has failed the provenance gate and not yet been rewritten
- WHEN a sync of `budget-planning-control-other-t1` is attempted
- THEN the sync MUST be blocked until `budget-planning-control` is rewritten and synced

### Requirement: `development` MUST have a minimal app manifest before any spec change is applied

Before `opsx-apply` runs against any synced financeq change, `development` MUST contain an
`appinfo/info.xml` declaring the six ADR-001 top-level menus (Begroting, Treasury, Subsidies,
Belastingen, Rapportage, Beheer) and a `src/manifest.json` skeleton with those six pages, per
ADR-024. Individual synced changes add their own widgets/sub-pages under the pre-declared parent
menu; no change may introduce a new top-level menu without an ADR-001 amendment.

#### Scenario: First applied change has a manifest to extend

- GIVEN `development` has the six-menu `appinfo/info.xml` and `src/manifest.json` skeleton from
  this change
- WHEN `opsx-apply` runs the first synced spec (e.g. `pc-cyclus-workflow`)
- THEN it extends the existing Begroting/Rapportage menu structure with the P&C-cyclus widget
- AND MUST NOT create a seventh top-level menu
