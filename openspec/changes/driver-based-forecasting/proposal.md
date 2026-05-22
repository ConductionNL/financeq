---
status: draft
version: 1.0
---

# Driver-Based Forecasting — Proposal

## Executive Summary

Driver-based forecasting predicts future costs and revenues from the *causal drivers* that produce them (inwoners, leerlingen, zorgvragers, kilometres of infrastructure, hectares groen) rather than from extrapolated time-series. The technique is standard in corporate FP&A for twenty years and increasingly mandated by NBA-handreiking 1108 for substantiating meerjarenramingen in Dutch decentrale overheden. Most gemeenten, provincies, and waterschappen still produce their meerjarenraming via Excel or time-series tools that cannot answer "what if we get 10% more inwoners in dit groeigebied". 

This spec provides the engine: driver register with provenance, cost/revenue formula DSL, scenario inheritance, rolling 18-month forecasts, variance analysis with attribution, what-if API, and sector reference packs. It integrates with `bookkeeping-bbv-compliance` (for realisatiecijfers), `pc-cyclus-workflow` (to promote scenarios into begroting-werkbestand), `decidesk` (to link policy decisions), `openconnector` (for CBS, PBL, DUO, Vektis feeds), and `mydash` (for dashboards).

## Features & Demand Scores

### Feature 1: Driver Registry with Provenance Tracking
**Demand: 9/10** (foundational, required by all downstream logic)

A clean register of forecast drivers with declared provenance (CBS-StatLine, BAG, Vektis, DUO, eigen waarnemingen), historical series, baseline forecast, and confidence intervals. Every driver declares refreshFrequency and maintains historicalSeries per provenance contract; system warns if staleness exceeds 2× refreshFrequency.

- Stakeholders: Concerncontroller, Strateeg, Analist
- Effort: Medium
- Risk: Data quality depends on upstream source contracts

### Feature 2: Typed DSL for Cost/Revenue Formulas
**Demand: 9/10** (core business logic)

A typed expression language with primitives `volume(driverCode)`, `kosten_per_eenheid(formulaConstant)`, `indexering(indexCode, component)`, `policy_override(scenarioParameterRef)`, and basic arithmetic. Formulas are statically type-checked at save time; unknown drivers surfaced with suggestions; potential divide-by-zero flagged.

- Stakeholders: Programma-controller, Strateeg
- Effort: High (parser + type checker + suggestion engine)
- Risk: Language design must handle future extensions without breaking

### Feature 3: Calibration from Historical Data
**Demand: 9/10** (accuracy and auditability)

Given a formula and N years of realisatiedata, calibrate kosten-per-eenheid via curve-fitting (RMSE, MAPE, R²). System creates explicit Calibration records; does not silently overwrite without acceptance. Low-confidence calibrations (MAPE > 25%) require documented justification.

- Stakeholders: Accountant, Concerncontroller, Analist
- Effort: Medium (relies on external stats library)
- Risk: Overfitting if historical data is sparse

### Feature 4: Scenario Inheritance & Parameter Overrides
**Demand: 8/10** (strategic flexibility)

Scenarios inherit parent parameters and override only what they specify. System detects conflicts (two parent sources override same driver-period with different values) and rejects with explanation. Policy measures reference decidesk besluiten; system warns if a besluit is verworpen.

- Stakeholders: Strateeg, Beleidsadviseur
- Effort: Medium (inheritance logic + conflict detection)
- Risk: Complexity grows with deep inheritance trees

### Feature 5: Rolling 18-Month Forecast Horizon
**Demand: 9/10** (operational requirement, BBV mandated)

Monthly schedule by default (configurable per RollingForecastSchedule). Each run covers 18 months (configurable). System retains prior runs (immutable history) and surfaces diff-view. Ad-hoc runs tagged separately.

- Stakeholders: Concerncontroller, Analist, Scheduler (n8n)
- Effort: Medium (scheduler integration)
- Risk: Dataset growth; older runs must be archived appropriately

### Feature 6: Variance Analysis with Attribution
**Demand: 9/10** (governance and learning)

For every closed period, compare most recent ForecastRun against actuals; decompose variance into volume-effect, prijs-effect, mix-effect, indexering-effect, onverklaard-residu. Unexplained variance > 5% triggers flag and assignment to formula owner for commentary. Commentary propagated to next run notes.

- Stakeholders: Accountant, Programma-controller, Analist
- Effort: High (decomposition algorithm, causal attribution)
- Risk: Attribution can be ambiguous; documentation critical

### Feature 7: What-If API
**Demand: 8/10** (strategic decision support)

Synchronous API: given base scenario + list of overrides (driver volume ±X%, kosten-per-eenheid ±Y%), returns computed ForecastRun deltas vs base per-formula, per-programma. Must complete within 2 seconds for typical workload (≤ 50 formulas × 18 months). Consumed by mydash and begroting-werkbestand workflow.

- Stakeholders: Strateeg, Beleidsadviseur, Controladviseur
- Effort: Medium (API optimization, caching)
- Risk: Performance regression under load

### Feature 8: Confidence Interval Propagation
**Demand: 7/10** (statistical rigor, not universally demanded)

When drivers carry confidence intervals, propagate through formulas via Monte Carlo (default 5000 samples); present forecast as central value + 90% confidence band. Drivers without declared CI treated as deterministic (flagged in notes).

- Stakeholders: Analist, Accountant
- Effort: Medium (MC simulation engine)
- Risk: Computational cost; 5000 samples may not be realistic for very complex models

### Feature 9: Integration into Begroting-Werkbestand
**Demand: 8/10** (policy integration)

Promote a ForecastRun (scenario × horizon) into the begroting-werkbestand of `pc-cyclus-workflow` as basis for that year's programmabegroting. Creates frozen snapshot (inputs, formulas, parameters); underlying drivers refresh independently (rolling forecast moves, begroting does not). Raads-amendementen captured as overrides in next-period variance.

- Stakeholders: Concerncontroller, Wethouder, Raad
- Effort: High (cross-app coordination, n8n orchestration)
- Risk: Governance complexity; must prevent accidental overwrites of frozen state

### Feature 10: Sector Reference Packs
**Demand: 7/10** (faster onboarding, not critical for initial launch)

Pre-built sets of Drivers + reference formulas + calibration starting points per sector (gemeente, provincie, waterschap) and per domeinen (sociaal domein, fysieke leefomgeving, jeugd, Wmo, onderwijs, wegen, riolering). New installations seed with matching pack(s); updates surface structured diff and require explicit confirmation.

- Stakeholders: Admin, VNG/IPO/UvW
- Effort: High (creation of 40+ reference packs, versioning strategy)
- Risk: Maintenance burden; packs must stay current with calibration research

## User Stories

### User Story 1: Concerncontroller promotes realistisch-2027 to begroting
**Persona:** Concerncontroller / hoofd Financiën  
**Frequency:** Weekly

As a Concerncontroller, I want to review a completed ForecastRun (e.g., realistisch-2027 covering 2027-2030) and promote it to the begroting-werkbestand so that the raad receives a forecast-backed meerjarenraming rather than a magic indexeringspercentage.

**GIVEN** a ForecastRun for scenario `realistisch-2027` covering 2027-2030 has been approved  
**WHEN** I click "Promote to Begroting 2027"  
**THEN** the system creates a frozen snapshot (drivers, formulas, parameters at run-time), links it to the 2027 begroting-werkbestand, and prevents the snapshot from changing even as underlying drivers refresh.

### User Story 2: Strateeg builds a groei-scenario and does what-if
**Persona:** Strateeg / corporate FP&A-rol  
**Frequency:** Daily (during planning cycles)

As a Strateeg, I want to create a child scenario `groeigebied-noord-realistisch-2027` (inheriting from `realistisch-2027`) with a +5% inwoners override for 2027-Q4, so that I can explore the budget impact of planned housing growth without creating a full new scenario from scratch.

**GIVEN** `realistisch-2027` scenario exists  
**WHEN** I create a child scenario with `inwoners-totaal` +5% in 2027-Q4  
**THEN** the system inherits all parent parameters, applies my override, and computes the ForecastRun in < 2 seconds via the what-if API.

### User Story 3: Programma-controller reviews variance and adds commentary
**Persona:** Programma-controller (Wmo, Jeugd, Onderwijs)  
**Frequency:** Bi-weekly (after period close)

As a Programma-controller for Wmo, I want to review the variance between my Q1-2027 forecast (EUR 12.5M) and actual (EUR 13.2M), understand whether the delta is due to more clients (volume-effect) or higher cao (prijs-effect), and add commentary so the analist can refine the formula for next cycle.

**GIVEN** a Variance record for Wmo budget Q1-2027 is available  
**WHEN** I view the variance detail page  
**THEN** the system decomposes the EUR 700k delta into volume-effect, prijs-effect, mix-effect, indexering-effect, and provides a form for my commentary.

### User Story 4: Analist notices driver is stale and refreshes
**Persona:** Analist / data steward  
**Frequency:** Monthly (ongoing)

As an Analist responsible for the driver register, I want to be notified when a driver (e.g., `inwoners-totaal` from CBS-StatLine) has not refreshed in > 2× its configured frequency, so that I can investigate and manually refresh if needed.

**GIVEN** `inwoners-totaal` is configured refreshFrequency=monthly and lastRefreshedAt > 2 months ago  
**WHEN** a ForecastRun references this driver  
**THEN** the system warns with `DRIVER_STALE`, includes the staleness in the run's confidence rationale, and displays a prominent notice to the Analist.

### User Story 5: Accountant audits calibration and accepts with rationale
**Persona:** Accountant / externe auditeur  
**Frequency:** Quarterly (during auditvoorbereiding)

As an Accountant, I want to review the Calibration records that underpin the cost formulas (e.g., kosten-per-wmo-client-zwaar = EUR 23,840) so that I can substantiate them for the accountant's report to the raad (NBA-handreiking 1108).

**GIVEN** a Calibration record with MAPE=18% (low-confidence flag)  
**WHEN** I view the calibration detail and review the RMSE/MAPE/R² metrics and the fitted constant  
**THEN** I can add a documented rationale (e.g., "accepted because historical data was stable 2023-2025") and mark it accepted, or reject it and request re-calibration.

## Stakeholder Profiles

### Concerncontroller / hoofd Financiën
- **Responsibility:** Promote scenarios to begroting, review rolling-forecast deltas, approve calibration changes, sign-off on what-if analyses for raad presentations.
- **Goals:** Ensure meerjarenraming is defensible, minimize unpleasant surprises in actualisation, support raad with data-backed scenario stories.
- **Weekly user.** Needs high-level dashboards (forecast vs actual per programma), easy promotion workflows, and confidence metrics.

### Strateeg / planner / corporate FP&A-rol
- **Responsibility:** Build and maintain scenarios, calibrate formulas, perform what-if for strategic heroverwegingen (budget cuts, growth initiatives, policy shifts).
- **Goals:** Rapid scenario iteration, transparent what-if results, documented parameter traceability (why is volume forecast +3% for inwoners in 2027-Q4?).
- **Primary daily user.** Needs powerful scenario editor, fast what-if API, scenario diff/comparison views, and export to Excel.

### Programma-controller (Wmo, Jeugd, Onderwijs, Openbare Ruimte)
- **Responsibility:** Own programma-specific formulas, review variance each period, update commentary explaining deltas, flag formulas that need re-calibration.
- **Goals:** Maintain formula accuracy, understand variance drivers, support budget adjustments, escalate anomalies to analist.
- **Bi-weekly user (period-close intensive).** Needs variance detail page, attribution breakdown, and easy commentary forms.

### Beleidsadviseur
- **Responsibility:** Use what-if sandbox to quantify financial impact of proposed policy variants (e.g., "what if we shift 15% of Jeugd budget to Wmo?"), present scenarios to portefeuillehouder.
- **Goals:** Transparent financials for policy trade-off discussions, avoid surprises when raad approves policies.
- **Occasional user (during policy cycles).** Needs user-friendly what-if UI, scenario comparison cards, export to slide decks.

### Wethouder / gedeputeerde / lid dagelijks bestuur
- **Responsibility:** Review scenario-comparison dashboards in sturingsgesprekken with raad/staten/AB, decide which scenario to adopt for meerjarenraming.
- **Goals:** Understand financial implications of strategic choices, communicate confidence and risk to raad.
- **Monthly/quarterly user.** Needs executive dashboards, scenario cards with key KPIs, and exported comparison PDFs.

### Accountant / externe auditeur
- **Responsibility:** Audit underpinnings of meerjarenraming (Calibration records, variance analyses, scenario logic), sign off on NBA substantiation.
- **Goals:** Verify formulas are grounded in history, confirm variance explanations, ensure confidence intervals are defensible.
- **Quarterly/annual user.** Needs full Calibration audit trail, variance commentary export, scenario documentation.

### Toezichthouder (provincie financieel toezicht, BZK)
- **Responsibility:** Monitor rolling forecasts as early-warning signal for sluitend meerjarenperspectief, request detail if variance anomalies flagged.
- **Goals:** Catch budget stress early, ensure sufficient reserves, identify systemic issues across sector.
- **Monthly user (reading published forecasts).** Needs anonymised aggregated sector trends, variance alerts.

### Raad / staten / AB
- **Responsibility:** Review and adopt meerjarenraming scenarios, amend programma-budgets, request scenario recalcs for amendments.
- **Goals:** Data-backed decision-making, transparency, confidence in multi-year planning.
- **Quarterly/annual user (decision moments).** Needs executive summary cards, scenario comparison, no technical jargon.

### VNG / IPO / UvW
- **Responsibility:** Collect and aggregate geanonimiseerde driver trends for sector prognoses (opt-in), feed into national macro-analysis.
- **Goals:** Understand sector-level trends, update reference packs, support peer benchmarking.
- **Annual user.** Needs data export with privacy controls, aggregation APIs.

### CPB / PBL
- **Responsibility:** Downstream consumer of sector-aggregated driver trends for macro-economic forecasting.
- **Goals:** Enrich national economic models with subnational detail, validate macro assumptions.
- **Annual user.** Needs aggregated data feeds, not individual registrations.

## Risks & Dependencies

- **Data quality:** System is only as good as the driver register. Staleness warnings and provenance tracking help, but require discipline from data stewards.
- **Formula complexity:** As more formulas are built, the DSL and documentation must stay readable. Peer review and reference packs help mitigate.
- **Integration with n8n:** Rolling forecast scheduler and promotion to begroting both use n8n. n8n must be stable and responsive.
- **Calibration accuracy:** MAPE and R² depend on stable historical data. Regime changes (e.g., new Wmo law, pandemic shock) may invalidate old calibrations.
- **Confidence interval propagation:** Monte Carlo is computationally expensive for large scenarios. Must carefully tune sample sizes.
- **Reference pack maintenance:** Creating 40+ sector packs is high-effort; updating them as law changes and calibrations improve is ongoing maintenance.

## Success Criteria

1. **Adoption:** By end of 2027, ≥ 30 Nederlandse decentrale overheden using driver-based forecasting for meerjarenraming.
2. **NBA compliance:** Accountants and toezichthouders can audit Calibration and Variance records to substantiate meerjarenraming per NBA-handreiking 1108.
3. **What-if fidelity:** Scenario deltas computed via what-if API match full ForecastRun within 0.1% for typical workloads.
4. **Performance:** Rolling-forecast scheduler completes monthly refresh within 5 minutes for 50+ scenarios × 8 entities × 18 months.
5. **Reference packs:** V1.0 includes gemeente, provincie, waterschap sector packs + Wmo/Jeugd/Onderwijs/OZB/Riolering domain packs, all calibrated from latest NBA/Vektis/CBS data.
