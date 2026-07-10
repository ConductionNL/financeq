## 1. Provenance gate (run before touching any branch)

- [ ] 1.1 For each of the 21 `origin/spec/*` branches, grep `openspec/changes/<name>/{proposal,specs,tasks,design}.md` for `Shillinq`/`shillinq` or any app id other than `financeq` appearing in the file's own title/summary (not in a cross-app dependency reference)
- [ ] 1.2 Confirm the 7-branch mislabeled list against HEAD of each branch (do not trust this proposal's list without re-checking — branches may have moved): `budget-planning-control`, `treasury-cash-management`, `tax-levy-management`, `obligation-financial-administration`, `financial-reporting-accountability`, `grant-subsidy-management`, `cost-accounting-allocation`
- [ ] 1.3 Confirm the 4 clean branches are still clean: `pc-cyclus-workflow`, `berap-marap-bestuursrapportages`, `iv3-aanlevering-cbs`, `driver-based-forecasting`

## 2. Sync the 4 clean root branches + bootstrap appinfo

- [ ] 2.1 Scaffold `appinfo/info.xml` on `development` per ADR-024 (app manifest) declaring the six ADR-001 top-level menus (Begroting, Treasury, Subsidies, Belastingen, Rapportage, Beheer) as navigation entries
- [ ] 2.2 Scaffold a minimal `src/manifest.json` with the six pages and no widgets (widgets/pages added per-spec by each change's own `opsx-apply`)
- [ ] 2.3 Copy `openspec/changes/pc-cyclus-workflow/` from `origin/spec/pc-cyclus-workflow` into `development` unchanged
- [ ] 2.4 Copy `openspec/changes/berap-marap-bestuursrapportages/` from `origin/spec/berap-marap-bestuursrapportages` unchanged
- [ ] 2.5 Copy `openspec/changes/iv3-aanlevering-cbs/` from `origin/spec/iv3-aanlevering-cbs` unchanged
- [ ] 2.6 Copy `openspec/changes/driver-based-forecasting/` from `origin/spec/driver-based-forecasting` unchanged
- [ ] 2.7 Convert each copied payload from the superseded specter format (monolithic `specs.md` + `hydra.json` + `context-brief.md`) to the house delta format: split `specs.md` into `specs/<capability>/spec.md` files using `## ADDED Requirements` + `### Requirement:` MUST-language + `#### Scenario:` GIVEN/WHEN/THEN blocks; keep `proposal.md`, `tasks.md` (all unchecked), `design.md`, and add `.openspec.yaml`
- [ ] 2.8 Run `openspec validate <name> --strict` for each of the 4 and resolve any errors

## 3. Rewrite the 7 mislabeled root branches for financeq's BBV domain

- [ ] 3.1 `budget-planning-control`: rewrite `proposal.md`/`specs.md`/`tasks.md`/`design.md` against ADR-001's Begroting scope (Productenraming, Investeringen, Mutaties per the t1-t3 placement table), replacing the generic multi-location/BOM procurement content
- [ ] 3.2 `treasury-cash-management`: rewrite against ADR-001's Treasury scope (Liquiditeitsprognose, Leningenportefeuille, Beleggingen, Schatkistbankieren)
- [ ] 3.3 `tax-levy-management`: rewrite against ADR-001's Belastingen scope (WOZ, Bezwaren, Heffingsverordeningen)
- [ ] 3.4 `obligation-financial-administration`: rewrite against gemeente verplichtingenadministratie (commitment accounting under BBV, not generic AP/procurement)
- [ ] 3.5 `financial-reporting-accountability`: rewrite against gemeente Rapportage scope (raads-rapportages, jaarrekening), cross-checked against the already-clean `berap-marap-bestuursrapportages` and `iv3-aanlevering-cbs` branches for domain consistency and to avoid overlap
- [ ] 3.6 `grant-subsidy-management`: rewrite against gemeente Subsidies scope (subsidieverordening, SiSa per ADR-001 §5), not generic grant-writing
- [ ] 3.7 `cost-accounting-allocation`: rewrite against gemeente kostenverdeling (cost-center/functie allocation under BBV, Beheer-owned drivers/tarieven per ADR-001 §5-6)
- [ ] 3.8 Author each rewrite directly in the house delta format (`specs/<capability>/spec.md` deltas + `.openspec.yaml`, per task 2.7), not the superseded monolithic `specs.md` format
- [ ] 3.9 Sync each rewritten root branch's `openspec/changes/<name>/` into `development` and run `openspec validate <name> --strict`

## 4. Defer the 10 tiered branches

- [ ] 4.1 File a follow-up change per parent module (`budget-planning-control-other-t1..t3`, `treasury-cash-management-other-t1..t4`, `tax-levy-management-other-t1..t3`) once its parent's rewrite (task 3) lands, so each tier inherits the corrected domain scope instead of being synced against the stale parent
- [ ] 4.2 Do not sync any tier branch in this change

## 5. Traceability

- [ ] 5.1 Cross-reference this change's synced specs against ADR-001's placement table to confirm every module lands under its assigned top-level menu
- [ ] 5.2 Record in `context-brief.md` (per synced change) which branch it was sourced from and whether it was copied unchanged or rewritten, for future audit
