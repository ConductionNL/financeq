# ADR-001: Information architecture for financeq

## Status
Proposed

## Date
2026-05-23

## Context

financeq is the gemeentelijke financial planning & control (P&C) suite:
BBV-conforme begroting, treasury & cash, subsidies, belastingen
(incl. WOZ), verplichtingen, kostenverdeling, and the verplichte
aanleveringen (Iv3 -> CBS, BERAP/MARAP, SiSa, jaarrekening). The repo
currently holds ~21 spec branches, several of them tier-suffixed
(`-other-t1..t4`) variants of a parent module: `budget-planning-control`
has three tiered children, `treasury-cash-management` has four,
`tax-levy-management` has three. Without a navigation contract these
tiered specs would naturally claim sibling menu items, producing the
ten-tabs-per-module "wall of menus" pattern that plagues legacy
gemeentepakketten (Centric, Pinkroccade, Cipers) and overwhelms the
gemeente CFO persona.

Compounding this, financeq is also the data backbone for two adjacent
apps: planix reads the BBV-programma-tree to ladder projects up to
programma's/doelen, and Rapportage joins the same tree for
raads-rapportages. If the tree is owned by a sub-page (or worse,
duplicated per module), every consumer breaks the moment a programma
is renamed.

A cross-app IA design session on 2026-05-22 (small-five doc:
financeq, purchaseq, planix, scholiq, openbuilt) produced an explicit
top-level menu structure for financeq and a set of placement rules
governing how the tiered specs, the P&C workflow, and the Iv3
configuration collapse into that structure. Those rules are
cross-cutting (they govern multiple specs, not the internals of any
single spec) and therefore belong in an ADR, not in any one spec.

This ADR captures the IA decisions so that future specs (and the
`team-architect` / `opsx-ff` / `app-apply` skills that scaffold them)
have a single navigation contract to honour. It does not redesign
any existing spec; it codifies the menu placement that the small-five
doc already assigns each spec.

## Decision

### 1. Top-level menus are capped at six

financeq exposes exactly six top-level menus, in this order:

1. **Begroting** (Budget Planning & Control)
2. **Treasury**
3. **Subsidies**
4. **Belastingen**
5. **Rapportage**
6. **Beheer**

No new top-level menu may be added without an ADR amendment. The
six-menu ceiling serves the concerncontroller / team-financien /
bestuurder personas, each of whom should reach their daily work in
one click.

### 2. Tier-suffixed specs collapse into the parent module as sub-pages

Specs named `{parent}-other-t1..t4` are detail branches of the parent
module and must be placed as sub-pages under the parent menu, never
as top-level menus and never as siblings of the parent. Concretely:

| spec | placement |
|---|---|
| `budget-planning-control-other-t1` | Begroting > Productenraming |
| `budget-planning-control-other-t2` | Begroting > Investeringen |
| `budget-planning-control-other-t3` | Begroting > Mutaties (tab) |
| `treasury-cash-management-other-t1` | Treasury > Liquiditeitsprognose |
| `treasury-cash-management-other-t2` | Treasury > Leningenportefeuille |
| `treasury-cash-management-other-t3` | Treasury > Beleggingen |
| `treasury-cash-management-other-t4` | Treasury > Schatkistbankieren |
| `tax-levy-management-other-t1` | Belastingen > WOZ |
| `tax-levy-management-other-t2` | Belastingen > Bezwaren |
| `tax-levy-management-other-t3` | Belastingen > Heffingsverordeningen |

This is what keeps the gemeente CFO menu under six items even though
the spec count is ~21.

### 3. BBV-programma-tree is the single source of truth

The BBV-programma-tree is owned by Begroting and read by every other
surface that needs to ladder up financially:

- Begroting writes it (Programmabegroting sub-page).
- Rapportage joins on it for raads-rapportages and BERAP/MARAP.
- planix consumes it (read-only) for project -> programma -> doel laddering.
- mydash rolls it up for bestuurders.

Renaming a programma or changing a budgetcode propagates everywhere;
no consumer is allowed to maintain its own copy. The tree lives in
openregister under the canonical schema published by financeq and
must not be duplicated in app-local state.

### 4. P&C-cyclus is a workflow status, not a menu

The P&C-cyclus (Kadernota -> Begroting -> Berap1 -> Berap2 -> Marap
-> Jaarrekening) is rendered as a breadcrumb + statusbalk widget
inside Begroting and Rapportage, with deep links to the relevant
documents. It must not become a seventh top-level menu and must not
be hidden inside Beheer; it is too operationally important to be
admin-only and too cross-cutting to live in one module.

`pc-cyclus-workflow` is therefore explicitly a cross-cutting widget
spec, not a page spec.

### 5. Iv3-taakvelden configuration lives in Beheer; aanlevering lives in Rapportage

The Iv3 taakvelden lookup is configuration data (rarely changes, set
once per gemeente, governed by CBS) and lives under Beheer alongside
COA, kostenplaatsen, functies, and tarieven. The act of validating
and submitting an Iv3 file to CBS is operational work (deadline-driven,
runs each kwartaal) and lives under Rapportage > Iv3-aanlevering CBS
as an action with validation preview. The user must never edit
taakvelden inside an aanlevering screen — config and submission are
separate surfaces.

The same separation applies to the other verplichte aanleveringen
(SiSa lives under Subsidies as a workflow, with regeling-templates
in Beheer; jaarrekening-export lives under Rapportage with sjablonen
in Beheer).

### 6. Adapters and connectors are hidden behind operational actions

Banking, CBS, SiSa, accountantssoftware, and any future external
integrations are configured under Beheer > Connectors and never
exposed as their own top-level menus or sub-pages. The user-facing
action that triggers an adapter call ("Cash sweepen", "Iv3 indienen",
"SiSa-bijlage genereren") hides the adapter — the begroter / treasurer
/ subsidiebeoordelaar never sees a "Connectors" surface in their
daily flow.

### 7. Menu visibility follows the persona, not the role-bit

Reguliere medewerkers (begroter, treasurer, subsidiebeoordelaar,
belastingmedewerker) see the five operational menus and do not see
Beheer. Concerncontrollers see all six. Bestuurders typically consume
via mydash and only drill into Rapportage. Menu visibility is wired
to persona, not to a raw permission bitmask, so a maker promoted to
controller does not need a separate login.

## Consequences

### Positive

- The gemeente CFO menu stays under six items even as the spec count
  grows past 21, matching the design ceiling that distinguishes
  financeq from legacy gemeentepakketten.
- Tiered specs (`-other-t1..t4`) have one canonical home; the
  team-architect skill can reject a proposal that tries to promote
  a tiered spec to a top-level menu without an ADR amendment.
- The BBV-programma-tree single-source rule prevents the most common
  rot mode in financial systems (programma renamed in budget, stale
  copy used in rapportage) and gives planix and mydash a stable
  read contract.
- Iv3 and other aanleveringen stay deadline-driven and operational,
  not buried under admin; CBS deadlines are visible where the work
  happens.

### Negative / cost

- Some specs that today read as "standalone modules" (e.g. the
  treasury tier-3 Beleggingen branch) lose the option of becoming
  their own menu. Future product asks to "promote Beleggingen to a
  top-level menu" require an ADR amendment, not just a PR.
- The Beheer menu becomes wide (COA, kostenplaatsen, taakvelden,
  functies, drivers, tarieven, periodes, rollen, connectors, logs).
  This is the deliberate trade-off for keeping the operational five
  narrow. Beheer is expected to use grouping sub-pages, not flat
  scroll.
- Cross-app reads (planix, mydash) must respect financeq as the
  authoritative writer of the BBV-tree; install-time deps and
  runtime contracts have to be specified per consumer (planix ADR
  already covers this on the planix side; mydash boundary covered
  by the existing "mydash must not depend on OR/openconnector at
  install time" rule — it reads the tree via runtime GraphQL only).

### Neutral

- The IA does not constrain the internals of any spec; it only
  constrains placement. Spec authors are free to design tabs,
  widgets, and actions inside a sub-page as long as the sub-page
  sits in the assigned parent menu.

## References

- Cross-app IA design doc (2026-05-22): financeq section in the
  small-five document covering financeq / purchaseq / planix /
  scholiq / openbuilt.
- ADR-022 (hydra): apps consume OR abstractions — supports the
  BBV-tree single-source rule.
- ADR-024 (hydra): app manifest — encodes the top-level menu
  contract per app.
- planix per-app IA ADR (sibling change): records the read-only
  consumer side of the BBV-tree contract.
