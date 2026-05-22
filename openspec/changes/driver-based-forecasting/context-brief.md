---
status: draft
---
# Driver-Based Forecasting

## Purpose

Driver-based forecasting predicts future costs and revenues from the *causal drivers* that produce them — number of inwoners, square metres of openbare ruimte, number of leerlingen in primair onderwijs, number of zorgvragers per zorgprofiel, kilometres of riolering, hectares groen — rather than from extrapolated time-series of historical financials. The technique has been standard in corporate FP&A (financial planning and analysis) for two decades and is increasingly mandated by accountancy guidance (NBA-handreiking 1108) for substantiating multi-year ramingen in the meerjarenraming of decentrale overheden. Yet most Dutch gemeenten, provincies, and waterschappen still produce their meerjarenraming via Excel sheets that apply an indexering-percentage to last year's number, or via traditional time-series tooling that cannot answer "what happens if we get 10% more inwoners in dit groeigebied".

The reasons are tooling and skills. Driver-based models need: a clean register of drivers with provenance (CBS, BAG, Vektis, DUO, eigen waarnemingen); per-driver kosten-per-eenheid and revenue-per-eenheid calibrated from history; per-driver volume forecasts (often from open data sources with their own uncertainty); scenario engines for optimistisch/realistisch/pessimistisch; rolling forecasts (refreshed monthly, not yearly); variance analysis comparing forecast vs actual; and a UI that lets a controller answer "wat als de Hervormingsagenda Jeugd 15% van het Wmo-budget verschuift" without writing code. Excel can do parts of this; nothing in the current decentrale-overheid stack ties it together.

`driver-based-forecasting` provides the engine. It owns: the register of forecast drivers (each with provenance, history, baseline forecast, and a confidence interval); the register of cost/revenue formulas (each linking a grootboek-rekening or programma-onderdeel to one or more drivers via a typed DSL); the scenario engine (named scenarios with parameter overrides — driver-volume +X%, kosten-per-eenheid -Y%, demografische schok, beleidsmaatregel); the rolling-forecast scheduler (monthly refresh, 18-month horizon); the variance-analysis view (forecast vs realised per driver, per programma, per cost-line); the what-if API consumed by mydash and by the begroting-werkbestand of `pc-cyclus-workflow`.

The spec does not own the underlying realisatiecijfers (those live in `bookkeeping-bbv-compliance`), the begroting itself (that is `programma-begroting`), nor the macro-economic indicators (those come via openconnector from CPB, CBS, Macrobond). It does not perform statistical estimation of confidence intervals — those come bundled with the driver from its source (e.g. CBS PBL prognose interval, eigen kalman-filter output) or are set ad-hoc by the analist with the rationale captured.

## Data Model

The spec introduces eight registers under `financeq/openspec/specs/driver-based-forecasting/schemas/`:

**Driver** — one record per forecast driver. Fields: `code` (e.g. `inwoners-totaal`, `wmo-clienten-65plus`, `leerlingen-po`, `oppervlakte-openbare-ruimte-m2`, `riool-km`, `bijstandsuitkeringen-aantal`), `naam`, `eenheid` (eenheid, m2, fte, km), `domain` (demografie|onderwijs|zorg|fysieke-leefomgeving|sociaal-domein|veiligheid|economie), `aggregationLevel` (organisation|wijk|buurt|gemeente-binnen-gr), `sourceRef` (openconnector source for refresh), `provenance` (CBS-StatLine-table-X|BAG|Vektis|DUO|eigen-meting|expert-judgement), `historicalSeries[]` (ordered time-points), `baselineForecastRef`, `lastRefreshedAt`, `refreshFrequency` (daily|monthly|quarterly|yearly).

**DriverForecast** — projected values for a Driver over a horizon. Fields: `driverRef`, `scenarioRef`, `forecastVersion`, `horizon` (e.g. 18 months, 4 years), `dataPoints[]` (ordered (date, value, lowerCI, upperCI)), `method` (cbs-prognose|trend-extrapolatie|ar-1|prophet|expert-judgement|policy-override), `methodParams`, `createdAt`, `createdBy`, `confidence` (high|medium|low + rationale).

**CostRevenueFormula** — links a financial target (grootboek-rekening, programma-onderdeel, taakveld) to one or more drivers via a typed DSL. Fields: `targetType` (grootboek-rekening|programma-onderdeel|taakveld|economische-categorie), `targetRef`, `formula` (DSL expression: e.g. `kosten_per_eenheid('wmo-zorgvraag-zwaar') * volume('wmo-clienten-zwaar') + indexering('cao-vng', 'salaris-component') * loonkosten_basis`), `driverRefs[]` (extracted from formula), `validFrom`, `validTo`, `calibrationRef` (link to last calibration run), `ownerRef`, `peerReviewedBy`, `documentation` (free text).

**Calibration** — derived from history: given a CostRevenueFormula and N years of realisatie, derive the kosten-per-eenheid that best fits. Fields: `formulaRef`, `period` (used for calibration), `derivedConstants[]` (e.g. kosten_per_wmo_client_zwaar_2024 = EUR 23,840), `rmse`, `mape`, `r2`, `notes`, `calibratedAt`.

**Scenario** — named what-if. Fields: `code` (e.g. `realistisch-2027`, `groeigebied-noord-2030`, `hervormingsagenda-jeugd`), `naam`, `description`, `parentScenarioRef` (allows scenario inheritance), `parameterOverrides[]` (list of (driverRef, period, multiplier|absoluteValue|deltaPct), or (formulaRef, constantName, override)), `policyMeasures[]` (links to decidesk besluiten that justify policy overrides), `tag` (optimistisch|realistisch|pessimistisch|stress|beleidsvariant).

**ForecastRun** — one execution of the engine for a (scenario × horizon). Fields: `scenarioRef`, `horizon`, `runAt`, `runBy` (user or scheduler), `inputSnapshots[]` (frozen Driver+Formula state), `outputs[]` (per target, per period: forecasted value, contributing drivers breakdown), `runtime`, `engineVersion`.

**Variance** — comparison of a ForecastRun against actuals as they materialise. Fields: `forecastRunRef`, `actualSource` (grootboek), `period`, `target`, `forecastValue`, `actualValue`, `delta`, `deltaPct`, `attribution[]` (decomposition of variance into volume-effect, prijs-effect, mix-effect, indexering-effect, onverklaard-residu), `commentary` (analist's note).

**RollingForecastSchedule** — configuration of the monthly refresh. Fields: `code`, `frequency` (daily|weekly|monthly|quarterly), `runDay`, `runTime`, `scenarioRefs[]` (which scenarios to refresh), `horizons[]`, `triggers[]` (calendar|event-realisatie-closed|event-driver-refreshed), `notificationRecipients[]`.

All registers reuse `core-financial-objects.Money` for monetary amounts and `core-time.PeriodSpec` for time-points. Confidence intervals use a standard `core-statistics.ConfidenceInterval` shape.

## Requirements

### REQ-001: Driver registry with provenance

Every Driver SHALL declare a verifiable provenance (sourceRef, refreshFrequency, lastRefreshedAt) and SHALL maintain a historicalSeries of at least the periods configured by its provenance contract (default 5 years).

- GIVEN a Driver `inwoners-totaal` with sourceRef pointing to CBS-StatLine-table-37230NED and refreshFrequency=monthly WHEN the monthly refresh runs THEN the system fetches the latest published value via openconnector and appends it to historicalSeries.
- GIVEN a Driver with provenance=expert-judgement WHEN it is created THEN the system requires a non-empty rationale field and a designated reviewer.
- GIVEN a Driver whose lastRefreshedAt is more than 2× refreshFrequency in the past WHEN any ForecastRun references it THEN the system warns with `DRIVER_STALE` and includes the staleness in the run's confidence rationale.

### REQ-002: Typed DSL for cost/revenue formulas

The system SHALL provide a typed expression DSL with explicit primitives `volume(driverCode)`, `kosten_per_eenheid(formulaConstant)`, `indexering(indexCode, component)`, `policy_override(scenarioParameterRef)`, and basic arithmetic; the DSL SHALL be statically validated at formula save time.

- GIVEN a formula `kosten_per_eenheid('wmo-zorg-zwaar') * volume('wmo-clienten-zwaar') + indexering('cao-vng', 'salaris-component') * 1850000` WHEN the user saves THEN the system parses, type-checks, extracts driverRefs, and persists.
- GIVEN a formula referencing an unknown driver `volume('typo-inwoners-totaal')` WHEN the user attempts to save THEN the system rejects with `UNKNOWN_DRIVER` and surfaces a "did you mean…" suggestion.
- GIVEN a formula that divides by `volume(...)` WHEN the static checker runs THEN it flags `POTENTIAL_DIVIDE_BY_ZERO` and requires the formula author to either add a guard or acknowledge.

### REQ-003: Calibration of constants from history

The system SHALL support calibration of formula constants from N years of realisatiedata, producing an explicit Calibration record with goodness-of-fit metrics, and SHALL not silently overwrite constants without an explicit calibration acceptance step.

- GIVEN a formula with constant `kosten_per_eenheid('wmo-zorg-zwaar')` and 4 years of realisatie WHEN the user runs calibration THEN the system fits the constant, returns the derived value with RMSE/MAPE/R², and waits for explicit acceptance before updating the formula.
- GIVEN a Calibration with MAPE > 25% WHEN it is presented THEN the system marks it `low-confidence` and requires a documented reason for acceptance.
- GIVEN an accepted Calibration WHEN the formula is updated THEN the prior constant value is captured in the formula's version history.

### REQ-004: Scenario inheritance and parameter overrides

The system SHALL support scenario inheritance (a child scenario inherits all parameters from its parent and overrides only what it specifies) and SHALL detect parameter conflicts (a child override that contradicts a sibling-merged source).

- GIVEN a scenario `realistisch-2027` and a child `groeigebied-noord-realistisch-2027` with override `volume('inwoners-totaal', 2027-Q4) *= 1.05` WHEN a ForecastRun executes for the child THEN it inherits all parent parameters and applies the +5% inwoners override.
- GIVEN two parent scenarios merged into a child WHEN they both override the same driver-period with different values THEN the system rejects with `SCENARIO_OVERRIDE_CONFLICT` listing the offending overrides.
- GIVEN a scenario with policyMeasures referencing a decidesk besluit in state `verworpen` WHEN a ForecastRun executes THEN the system warns and requires the scenario to be archived or re-tagged.

### REQ-005: Rolling 18-month forecast horizon

The system SHALL produce, on a monthly schedule by default, a rolling 18-month forecast for every active scenario; the horizon SHALL be configurable per RollingForecastSchedule but SHALL default to 18 months.

- GIVEN a RollingForecastSchedule configured monthly-on-day-5 with horizon=18 months WHEN it runs on 2027-04-05 THEN the system produces a ForecastRun covering 2027-04 to 2028-09 inclusive.
- GIVEN a previous monthly ForecastRun for 2027-03 WHEN the 2027-04 run completes THEN the system retains the prior run (immutable history) and surfaces a diff-view.
- GIVEN a manual ad-hoc ForecastRun triggered outside the schedule WHEN it completes THEN the system tags it `ad-hoc` and excludes it from the rolling-history view by default.

### REQ-006: Variance analysis with attribution

For every closed period, the system SHALL compute Variance records comparing the most recent ForecastRun (made before period close) against actuals and SHALL decompose variance into volume-effect, prijs-effect, mix-effect, indexering-effect, and onverklaard-residu.

- GIVEN a forecast for Q1-2027 Wmo-zorg-zwaar of EUR 12,500,000 and actuals of EUR 13,200,000 WHEN variance is computed THEN the system decomposes: volume-effect (clienten +3%) = EUR 375,000, prijs-effect (cao boven verwachting) = EUR 220,000, mix-effect = EUR 80,000, residu = EUR 25,000.
- GIVEN a variance with onverklaard-residu > 5% van forecast WHEN it is shown on the dashboard THEN the system flags it `requires-explanation` and assigns it to the formula's owner.
- GIVEN a variance commentary added by the owner WHEN the next ForecastRun executes THEN the commentary is propagated as context to the run notes.

### REQ-007: What-if API

The system SHALL expose a synchronous what-if API that, given a base scenarioRef and a list of overrides, returns a computed ForecastRun result without persisting; the API SHALL complete within 2 seconds for a typical (≤ 50 formulas × 18 months) workload.

- GIVEN a POST /api/forecast/what-if with base=realistisch-2027 and overrides=`[{driver: 'inwoners-totaal', period: '2027-Q4', deltaPct: 5}]` WHEN the request executes THEN the system returns per-formula deltas vs base and total delta per programma within 2 seconds.
- GIVEN a what-if request with overrides referencing a deprecated driver WHEN it executes THEN the system returns 400 with `DEPRECATED_DRIVER` and a suggested successor.
- GIVEN a what-if request that times out (e.g. very large scenario tree) WHEN the limit is hit THEN the system returns 503 with `WHATIF_TIMEOUT` and a recommendation to use the async ForecastRun endpoint.

### REQ-008: Confidence interval propagation

When drivers carry confidence intervals, the system SHALL propagate them through the formulas using Monte Carlo simulation (default 5000 samples) and SHALL present the forecast as a central value with a 90% confidence band.

- GIVEN a driver `wmo-clienten-zwaar` with central forecast 2,400 and 90% CI [2,250 – 2,580] WHEN a ForecastRun executes formula `kosten_per_eenheid * volume` THEN the system Monte Carlo's the kosten output and surfaces central + 90% band.
- GIVEN a Monte Carlo run with fewer than 5000 samples due to runtime constraint WHEN it completes THEN the system warns `LOW_SAMPLE_COUNT` and surfaces the actual sample count.
- GIVEN a driver with no declared CI WHEN it enters a formula THEN the system treats its uncertainty as zero and notes `assumed-deterministic` in the run rationale.

### REQ-009: Integration into begroting-werkbestand

The system SHALL allow a ForecastRun (specific scenario, specific horizon) to be promoted into the begroting-werkbestand of `pc-cyclus-workflow` as the basis for that year's programmabegroting; the promotion creates a frozen snapshot.

- GIVEN a ForecastRun for scenario `realistisch-2027` covering 2027-2030 WHEN the concerncontroller promotes it to the begrotings-werkbestand 2027 THEN the system freezes a snapshot (inputs, formulas, parameters) and links the begroting-regels to the snapshot.
- GIVEN a promoted ForecastRun WHEN the underlying drivers later refresh THEN the begroting-werkbestand snapshot does NOT change (frozen) but the rolling-forecast keeps moving and the dashboard surfaces the delta.
- GIVEN a begroting-werkbestand 2027 promoted from `realistisch-2027` WHEN the raad amendes a programma-budget upward THEN the deviation is captured as a `raads-amendement` override and surfaced in next-period variance attribution.

### REQ-010: Sector reference packs

The system SHALL ship reference packs (sets of Drivers + reference formulas + calibration starting points) per sector: gemeente, provincie, waterschap, and per dominante domeinen: sociaal domein, fysieke leefomgeving, jeugd, Wmo, onderwijs, wegen, riolering.

- GIVEN a fresh financeq installation for Gemeente Zeist WHEN the user runs `seed-data driver-based-forecasting:reference-gemeente` THEN the system imports the gemeente driver-pack, the Wmo/Jeugd/Onderwijs formula packs, and Q4-2024 calibration starting points.
- GIVEN a reference pack version 2026.1 already imported and version 2027.1 published WHEN the user updates THEN the system surfaces a structured diff (new drivers, renamed drivers, formula changes) and requires explicit confirmation before applying.
- GIVEN a waterschap installation WHEN the gemeente pack is offered THEN the system filters it out and recommends the waterschap pack instead.

## Standards & Sources

- **NBA-handreiking 1108** — accountantsprotocol that increasingly references substantiation of meerjarenraming via driver logic.
- **BBV** — meerjarenraming structure (3-jaars-horizon na begrotingsjaar) within which 18-month rolling forecasts inform raming-bijstellingen.
- **CBS-prognoses** — `Prognose huishoudens 2024-2070` (PBL/CBS), `Bevolkingsprognose 2024-2070`, `Onderwijs-leerlingenprognoses` (DUO).
- **Vektis** — zorg-realisatiedata onderbouwen kosten-per-eenheid voor Wmo en Jeugdwet.
- **DUO** — leerling-aantallen per gemeente per onderwijs-soort.
- **BAG** — woon-, niet-woon-objecten als drivers voor OZB-baten en afval-baten.
- **CPB** — macro-economische verkenning en CEP voor indexering-aannames (cao-loonkostenontwikkeling, prijsmutatie BBP).
- **PBL** — scenario-aannames demografie en ruimtelijke ontwikkeling.
- **VNG / IPO / UvW** — sector-benchmarks per programma voor cross-check.
- **ESA 2010 / Wet Hof** — saldobegrenzing geeft macro-randvoorwaarden voor scenario-kiezen.
- **Reference systems** — Pepperflow Forecast (gemeente-markt), Vena (corporate FP&A), Anaplan, Workday Adaptive Planning, IBM Cognos Planning Analytics, Cube. The spec is interoperable but not derived from any.
- **Academic / methodological** — Hyndman & Athanasopoulos "Forecasting: Principles and Practice" (open access) is the methodological reference for the bundled extrapolation methods.

## Cross-app integration

- **bookkeeping-bbv-compliance** (financeq) — supplies the grootboek-rekeningen targeted by formulas and the realisatiecijfers used in Calibration and Variance.
- **bookkeeping-budget-forecast** (financeq) — consumes the ForecastRun outputs as input for the meerjarenraming sheets.
- **pc-cyclus-workflow** (financeq) — REQ-009 promotes scenarios into the begroting-werkbestand; rolling forecasts feed BERAP variance-analyses.
- **iv3-aanlevering-cbs** (financeq) — raming-aanleveringen consume the latest promoted scenario.
- **decidesk** — policy-override scenarios link to raadsbesluiten; a verworpen besluit retires its scenarios.
- **openconnector** — sources for CBS-StatLine, PBL prognoses, DUO leerling-aantallen, Vektis Wmo-realisatie, CPB macro-indicators, BAG.
- **openregister** — every register inherits OR audit-trail, versioning, search, and access control.
- **n8n** — rolling-forecast scheduler runs as n8n workflow; variance-flag notifications routed via n8n; scenario promotion to begroting routed via n8n for sign-off.
- **mydash** — exposes a "Forecast vs Actual per programma" widget, a "Scenario Comparison" widget, and a what-if sandbox.
- **docudesk** — archives promoted ForecastRun snapshots together with the begroting-werkbestand they substantiate.
- **docusaurus journeydoc** — ships a "Wmo-budget onderbouwen met driver-based" how-to and a "What-if doen: hervormingsagenda jeugd" walkthrough.

## Target users

- **Concerncontroller / hoofd Financiën** — promotes scenarios to begroting, reviews rolling-forecast deltas, approves calibration changes. Weekly user.
- **Strateeg / planner / corporate FP&A-rol** — bouwt en onderhoudt scenario's, doet what-if voor strategische heroverwegingen. Primary daily user.
- **Programma-controller (Wmo, Jeugd, Onderwijs, Openbare Ruimte)** — eigenaar van programma-specifieke formules, beoordeelt variance, voegt commentaar toe.
- **Beleidsadviseur** — gebruikt het what-if scherm om financiële impact van beleidsvarianten te tonen aan de portefeuillehouder.
- **Wethouder / gedeputeerde / lid dagelijks bestuur** — consumeert scenario-vergelijking-dashboards in het sturingsgesprek met de raad/staten/AB.
- **Accountant** — toetst onderbouwing van meerjarenraming via Calibration records en variance-historie.
- **Toezichthouder (provincie financieel toezicht, BZK)** — leest de rolling-forecast als early-warning signaal voor sluitend meerjarenperspectief.
- **Raad / staten / AB** — krijgt scenario-onderbouwing bij meerjarenraming i.p.v. een ondoorzichtig indexeringspercentage.
- **VNG / IPO / UvW** — aggregeert geanonimiseerde driver-trends voor sector-prognoses (opt-in).
- **CPB / PBL** — downstream consument van sector-aggregaten voor macro-analyse.
