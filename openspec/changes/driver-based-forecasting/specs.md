---
status: draft
version: 1.0
---

# Driver-Based Forecasting — Specifications

## REQ-001: Driver Registry with Provenance

**Requirement:** Every Driver SHALL declare a verifiable provenance (sourceRef, refreshFrequency, lastRefreshedAt) and SHALL maintain a historicalSeries of at least the periods configured by its provenance contract (default 5 years).

### REQ-001-001: Scheduled driver refresh from CBS-StatLine
**GIVEN** a Driver `inwoners-totaal` with:
- sourceRef = `cbs-statline-37230NED`
- refreshFrequency = `monthly`
- lastRefreshedAt = `2026-04-30T10:00:00Z`

**WHEN** the monthly refresh (e.g., 2026-05-05 at 05:00) is triggered via RollingForecastSchedule

**THEN**:
- openconnector fetches the latest published value from CBS-StatLine table 37230NED
- System appends (period, value) to Driver.historicalSeries
- lastRefreshedAt is updated to run timestamp
- historicalSeries contains ≥ 5 years of prior periods
- An AuditTrail entry logs the refresh (user='system', action='refresh', before/after snapshots)

### REQ-001-002: Staleness warning on ForecastRun
**GIVEN** a Driver with:
- refreshFrequency = `quarterly`
- lastRefreshedAt = `2025-11-01T09:00:00Z` (6 months in the past)

**WHEN** a ForecastRun references this driver (e.g., 2026-05-05)

**THEN**:
- System detects lastRefreshedAt > 2 × refreshFrequency (6 months > 2 × 3 months)
- ForecastRun.confidence.warnings includes:
  - code: `DRIVER_STALE`
  - message: `Driver inwoners-totaal last refreshed 2025-11-01; expected quarterly refresh (overdue 2 months)`
- ForecastRun.confidence.overallLevel is downgraded (high → medium, or medium → low)
- Dashboard surfaces prominent warning to Analist

### REQ-001-003: Expert-judgement driver requires rationale
**GIVEN** a new Driver with:
- provenance = `expert-judgement`

**WHEN** user attempts to save without providing confidenceRationale

**THEN**:
- System rejects with validation error: `EXPERT_JUDGEMENT_REQUIRES_RATIONALE`
- confidenceRationale field must be non-empty before save succeeds

### REQ-001-004: Historical series validation
**GIVEN** a Driver being saved

**WHEN** historicalSeries contains fewer periods than configured by provenance contract (e.g., refreshFrequency=monthly and only 2 years of data)

**THEN**:
- System warns but allows save (not a hard blocker)
- Warning message: `Historical series contains 24 periods, but refreshFrequency=monthly suggests ≥60 periods for 5-year standard`
- Dashboard flags for Analist attention

---

## REQ-002: Typed DSL for Cost/Revenue Formulas

**Requirement:** The system SHALL provide a typed expression DSL with explicit primitives `volume(driverCode)`, `kosten_per_eenheid(formulaConstant)`, `indexering(indexCode, component)`, `policy_override(scenarioParameterRef)`, and basic arithmetic; the DSL SHALL be statically validated at formula save time.

### REQ-002-001: DSL parsing and validation
**GIVEN** a CostRevenueFormula with:
- formula = `kosten_per_eenheid('wmo-zorg-zwaar') * volume('wmo-clienten-zwaar') + indexering('cao-vng', 'salaris-component') * 1850000`

**WHEN** user saves the formula

**THEN**:
- Parser tokenizes and builds an AST
- Type checker validates:
  - volume('wmo-clienten-zwaar') references an existing Driver with code='wmo-clienten-zwaar'
  - kosten_per_eenheid('wmo-zorg-zwaar') references a known constant (from Calibrations or prior definitions)
  - indexering('cao-vng', 'salaris-component') references a known index+component pair
  - Result type is compatible with targetType (e.g., Money for großbuch-rekening)
- driverRefs are extracted and auto-populated: ["wmo-clienten-zwaar"]
- Formula is persisted

### REQ-002-002: Unknown driver detection with suggestions
**GIVEN** a formula with:
- formula = `volume('typo-inwoners-totaal') * 1000`

**WHEN** user saves

**THEN**:
- Parser identifies unknown driver code: 'typo-inwoners-totaal'
- System searches existing drivers for similar codes (Levenshtein distance)
- Validation error returned with suggestions:
  - code: `UNKNOWN_DRIVER`
  - message: `Driver 'typo-inwoners-totaal' not found`
  - suggestions: `["inwoners-totaal", "inwoners-werkers", "inwoners-jong"]`
- Save is rejected; user must correct or confirm intent

### REQ-002-003: Divide-by-zero flag
**GIVEN** a formula with:
- formula = `1000000 / volume('bevolking')`

**WHEN** user saves

**THEN**:
- Static analyzer detects potential divide-by-zero: volume('bevolking') in denominator
- Validation returns warning (not error):
  - code: `POTENTIAL_DIVIDE_BY_ZERO`
  - message: `Denominator contains volume(...); may be zero or negative`
- User must explicitly acknowledge (checkbox: "I have verified denominators cannot be zero") before save succeeds

### REQ-002-004: Arithmetic operator support
**GIVEN** DSL parsing

**WHEN** parsing expressions with operators: `+`, `-`, `*`, `/`, `%`, `^` (exponentiation)

**THEN**:
- All operators parse and type-check correctly
- Operator precedence follows standard math (^ > {*,%,/} > {+,-})
- Parentheses override precedence
- Expression `volume('x') * kosten_per_eenheid('y') + indexering('index', 'comp') * base` is parsed as `(volume('x') * kosten_per_eenheid('y')) + (indexering(...) * base)`

---

## REQ-003: Calibration from Historical Data

**Requirement:** The system SHALL support calibration of formula constants from N years of realisatiedata, producing an explicit Calibration record with goodness-of-fit metrics, and SHALL not silently overwrite constants without an explicit calibration acceptance step.

### REQ-003-001: Calibration execution and acceptance workflow
**GIVEN**:
- CostRevenueFormula with constant placeholder `kosten_per_eenheid('wmo-zorg-zwaar')`
- 4 years of realisatiedata (2023-2026) from bookkeeping-bbv-compliance

**WHEN** user clicks "Run Calibration" on the formula detail page

**THEN**:
- System fetches realisatie for all periods in range
- Curve-fit algorithm (least squares or similar) derives optimal constant value
- Calibration object created with:
  - formulaRef = this formula
  - period = "2023-2026"
  - derivedConstants = [{ constantName: 'wmo-zorg-zwaar', value: EUR 23,840, confidence: 'high' }]
  - rmse = 1200, mape = 4.3, r2 = 0.94
- Calibration.acceptanceStatus = 'pending'
- User is shown a detail form with metrics, fitted value, and prior value for comparison
- Formula is NOT updated; awaiting acceptance

### REQ-003-002: Low-confidence calibration requires rationale
**GIVEN** a Calibration with:
- mape = 26.5% (exceeds 25% threshold)

**WHEN** system computes and presents the calibration

**THEN**:
- Calibration.acceptanceStatus defaults to 'pending'
- System marks it with confidence flag: `low-confidence`
- UI displays prominent warning: "MAPE > 25% indicates poor fit; requires documented justification for acceptance"
- acceptanceRationale field is required (non-empty) for acceptance
- If user attempts to accept without rationale, form rejects with `RATIONALE_REQUIRED`

### REQ-003-003: Calibration acceptance and versioning
**GIVEN** a pending Calibration with acceptanceStatus='pending'

**WHEN** user provides acceptanceRationale and clicks "Accept Calibration"

**THEN**:
- System updates Calibration.acceptanceStatus = 'accepted'
- Calibration.acceptedBy = authenticated user ID
- Calibration.acceptedAt = current timestamp
- CostRevenueFormula.calibrationRef is updated to point to this Calibration
- Prior constant value in formula is captured in formula's version history (AuditTrail records old value before/after snapshot)
- New constant value is active in next ForecastRun

### REQ-003-004: Calibration rejection
**GIVEN** a Calibration with poor metrics (e.g., mape > 40%)

**WHEN** user clicks "Reject Calibration"

**THEN**:
- Calibration.acceptanceStatus = 'rejected'
- Calibration.acceptanceRationale is captured (e.g., "Regime change due to Wmo law change in 2025; historical data not representative")
- Formula is NOT updated; formula owner must either re-calibrate or manually adjust constant
- Rejection is logged in AuditTrail

---

## REQ-004: Scenario Inheritance & Parameter Overrides

**Requirement:** The system SHALL support scenario inheritance (a child scenario inherits all parameters from its parent and overrides only what it specifies) and SHALL detect parameter conflicts (a child override that contradicts a sibling-merged source).

### REQ-004-001: Scenario inheritance and parameter merging
**GIVEN**:
- Scenario `realistisch-2027` (parent) with parameterOverrides = []
- Creating child Scenario `groeigebied-noord-realistisch-2027` with:
  - parentScenarioRef = `realistisch-2027`
  - parameterOverrides = [{ type: 'driver-volume', driverRef: 'inwoners-totaal', period: '2027-Q4', overrideType: 'multiplier', value: 1.05 }]

**WHEN** a ForecastRun is triggered for child scenario

**THEN**:
- Engine resolves child parameters: child.parameterOverrides ∪ parent.parameterOverrides (child overrides parent if both specify same (driverRef, period))
- ForecastRun executes with merged parameters
- Driver 'inwoners-totaal' in 2027-Q4 is multiplied by 1.05 (inherited from child)
- All other parameters inherited from parent

### REQ-004-002: Scenario conflict detection
**GIVEN**:
- Parent scenario `realistisch-2027` with override: `inwoners-totaal` 2027-Q4 *= 1.00 (no change)
- Child scenario with two parents (not linear inheritance, but multi-parent merge):
  - Parent A: `inwoners-totaal` 2027-Q4 *= 1.05
  - Parent B: `inwoners-totaal` 2027-Q4 *= 1.02 (conflicting value)

**WHEN** system attempts to merge parameters

**THEN**:
- System detects conflict: two parent sources specify different overrides for same (driverRef, period)
- Validation returns error:
  - code: `SCENARIO_OVERRIDE_CONFLICT`
  - message: `Conflicting overrides for inwoners-totaal (2027-Q4): Parent A=×1.05, Parent B=×1.02`
- Child scenario save is rejected; user must resolve (choose one parent's value, or explicitly override)

### REQ-004-003: Policy measure linkage to decidesk
**GIVEN** a Scenario with:
- policyMeasures = [{ decidesk_besluitRef: 'rb-2025-12-woningbouw-noord', rationale: '...' }]

**WHEN** a ForecastRun executes for this scenario

**THEN**:
- System queries decidesk for besluit 'rb-2025-12-woningbouw-noord' to check status
- If status = 'aangenomen' or 'geeffectueerd': continue normally
- If status = 'verworpen' or 'ingetrokken': ForecastRun.confidence.warnings includes:
  - code: `POLICY_MEASURE_VERWORPEN`
  - message: `Policy measure 'woningbouw-noord' (rb-2025-12-woningbouw-noord) has status verworpen; scenario may be outdated`
- Scenario is flagged for archival or re-tagging

---

## REQ-005: Rolling 18-Month Forecast Horizon

**Requirement:** The system SHALL produce, on a monthly schedule by default, a rolling 18-month forecast for every active scenario; the horizon SHALL be configurable per RollingForecastSchedule but SHALL default to 18 months.

### REQ-005-001: Rolling forecast monthly execution
**GIVEN** RollingForecastSchedule with:
- frequency = 'monthly'
- runDay = 5
- runTime = '05:00'
- horizon = '18 months'

**WHEN** 2027-04-05 at 05:00 UTC arrives

**THEN**:
- System triggers ForecastRun for all scenarioRefs in the schedule
- ForecastRun.horizon = '18 months'
- ForecastRun.outputs covers periods: 2027-04, 2027-05, ..., 2028-09 (18 consecutive months)
- ForecastRun.runType = 'rolling'
- ForecastRun.runBy = 'system'
- ForecastRun.status = 'completed' (or 'failed' with error message)

### REQ-005-002: Previous runs retained for history
**GIVEN**:
- Previous ForecastRun for 2027-03 (18 months, covering 2027-03 to 2028-08)
- New ForecastRun for 2027-04 (18 months, covering 2027-04 to 2028-09)

**WHEN** 2027-04 run completes

**THEN**:
- Both ForecastRun objects remain in database (immutable history)
- Rolling-forecast dashboard can display both runs side-by-side
- Diff-view shows what changed from 2027-03 to 2027-04 (new 2028-09, old 2027-03 dropped, mid-period updates)
- Older runs can be archived (moved to cold storage) after retention period (e.g., 2 years)

### REQ-005-003: Ad-hoc runs excluded from rolling history
**GIVEN** user manually triggers ForecastRun for scenario `realistisch-2027` outside the scheduled window

**WHEN** ForecastRun completes

**THEN**:
- ForecastRun.runType = 'ad-hoc'
- Rolling-forecast dashboard displays rolling runs by default
- Ad-hoc runs are accessible via filter or archived/historical view
- Notification recipients are notified, but run is not included in standard rolling-forecast metrics

---

## REQ-006: Variance Analysis with Attribution

**Requirement:** For every closed period, the system SHALL compute Variance records comparing the most recent ForecastRun (made before period close) against actuals and SHALL decompose variance into volume-effect, prijs-effect, mix-effect, indexering-effect, and onverklaard-residu.

### REQ-006-001: Variance decomposition
**GIVEN**:
- ForecastRun for Q1-2027 Wmo-zorg-zwaar: EUR 12,500,000
- Actual (from bookkeeping-bbv-compliance) Q1-2027 Wmo-zorg-zwaar: EUR 13,200,000
- Delta: EUR 700,000 (+5.6%)

**WHEN** variance computation runs after Q1-2027 period close (e.g., 2027-04-15)

**THEN**:
- System retrieves actuals from bookkeeping and creates Variance record
- Attribution algorithm decomposes delta:
  - volume-effect (clienten +3%): EUR 375,000
  - prijs-effect (cao boven verwachting): EUR 220,000
  - mix-effect (client distribution shift): EUR 80,000
  - indexering-effect: EUR 0
  - onverklaard-residu: EUR 25,000
- Variance.attribution = array of { type, amount, percentOfDelta } tuples
- Sum of attribution amounts ≈ Variance.delta (within rounding tolerance)

### REQ-006-002: High-variance flag and assignment
**GIVEN** a Variance with |deltaPct| > 5%

**WHEN** variance is computed

**THEN**:
- Variance.requiresExplanation = true
- Variance is automatically assigned to CostRevenueFormula.ownerRef (programma-controller)
- Notification sent: "Variance for [target] [period] requires explanation: delta = [amount] ([pct]%)"
- Dashboard surfaces unexplained variance prominently (flagged red)

### REQ-006-003: Owner commentary propagation
**GIVEN** a Variance with requiresExplanation = true

**WHEN** formula owner adds commentary and saves Variance.commentaryByOwner

**THEN**:
- Variance.commentaryAddedAt = current timestamp
- Variance.commentaryAddedBy = authenticated user ID
- Variance.flaggedFor = 'resolved' (if commentary is satisfactory)
- Next monthly ForecastRun for same target includes variance commentary in ForecastRun.notes or context
- Accountant can review variance + commentary together for audit trail

---

## REQ-007: What-If API

**Requirement:** The system SHALL expose a synchronous what-if API that, given a base scenarioRef and a list of overrides, returns a computed ForecastRun result without persisting; the API SHALL complete within 2 seconds for a typical (≤ 50 formulas × 18 months) workload.

### REQ-007-001: What-if API endpoint
**GIVEN** a POST request to `/api/forecast/what-if` with body:
```json
{
  "baseScenarioRef": "realistisch-2027",
  "overrides": [
    {
      "type": "driver-volume",
      "driverRef": "inwoners-totaal",
      "period": "2027-Q4",
      "overrideType": "deltaPct",
      "value": 5
    }
  ]
}
```

**WHEN** request is received

**THEN**:
- API fetches base scenario and its latest ForecastRun
- Applies overrides (merged with base scenario overrides)
- Executes formula engine in-memory
- Returns response (not persisted in ForecastRun table):
```json
{
  "baseScenarioRef": "realistisch-2027",
  "overrides": [...],
  "outputs": [
    {
      "targetRef": "wmo-zorg-zwaar-2027",
      "period": "2027-Q4",
      "forecastValue": { "amount": 12,345,000, "currency": "EUR" },
      "deltaVsBase": { "amount": 450,000, "currency": "EUR" },
      "deltaPctVsBase": 3.8
    }
  ],
  "computedAt": "2027-04-15T14:23:45Z",
  "runtime": 1200  // milliseconds
}
```
- Response time ≤ 2 seconds

### REQ-007-002: Deprecated driver detection
**GIVEN** a what-if request with override:
```json
{
  "type": "driver-volume",
  "driverRef": "inwoners-oud",  // deprecated driver
  "period": "2027-Q4",
  "overrideType": "multiplier",
  "value": 1.05
}
```

**WHEN** API processes overrides

**THEN**:
- System checks if driver is deprecated (marked with validTo < current date or deprecatedReplacementRef set)
- Returns 400 Bad Request:
```json
{
  "code": "DEPRECATED_DRIVER",
  "message": "Driver 'inwoners-oud' is deprecated",
  "suggestion": "Use 'inwoners-totaal' instead"
}
```

### REQ-007-003: Timeout on complex scenarios
**GIVEN** a what-if request for a very large scenario tree (200+ formulas, 48-month horizon) that would exceed time budget

**WHEN** engine is processing and approaches 2-second limit

**THEN**:
- System monitors execution time
- If runtime approaches 2000ms, engine may curtail simulation (reduce sample count for Monte Carlo, skip lower-priority outputs)
- If curtailment insufficient and engine would exceed 2 seconds: return 503 Service Unavailable:
```json
{
  "code": "WHATIF_TIMEOUT",
  "message": "Forecast computation timed out; scenario is too complex for synchronous evaluation",
  "suggestion": "Use async ForecastRun endpoint instead"
}
```

---

## REQ-008: Confidence Interval Propagation

**Requirement:** When drivers carry confidence intervals, the system SHALL propagate them through the formulas using Monte Carlo simulation (default 5000 samples) and SHALL present the forecast as a central value with a 90% confidence band.

### REQ-008-001: Monte Carlo simulation for confidence intervals
**GIVEN**:
- Driver `wmo-clienten-zwaar` with central forecast 2,400 and 90% CI [2,250 – 2,580]
- Formula: `kosten_per_eenheid('wmo-zorg-zwaar') * volume('wmo-clienten-zwaar')`
- kosten_per_eenheid = EUR 10,000 (deterministic, no CI)

**WHEN** ForecastRun executes

**THEN**:
- System samples volume('wmo-clienten-zwaar') 5,000 times from CI distribution (assumed uniform or normal with bounds)
- For each sample, computes formula output
- Extracts central (median), lower 5th percentile, upper 95th percentile from 5,000 outputs
- ForecastRun.outputs includes:
```json
{
  "targetRef": "wmo-zorg-zwaar-2027",
  "period": "2027-Q1",
  "forecastValue": { "amount": 24,000,000, "currency": "EUR" },
  "confidenceInterval": {
    "lowerBound": 22,500,000,
    "upperBound": 25,800,000,
    "methodology": "monte-carlo-5000-samples"
  }
}
```

### REQ-008-002: Multiple drivers with confidence intervals
**GIVEN**:
- Driver A with CI [100–110]
- Driver B with CI [50–60]
- Formula: `volume(A) * kosten_per_eenheid * volume(B)`

**WHEN** Monte Carlo executes

**THEN**:
- System samples both drivers independently
- Combines samples through formula (correlated or independent, depends on formula structure)
- Final 90% CI reflects joint uncertainty from both drivers

### REQ-008-003: Deterministic drivers flagged
**GIVEN** a Driver without declared confidence intervals (lowerCI, upperCI both null)

**WHEN** formula uses this driver

**THEN**:
- ForecastRun.confidence.warnings includes:
  - code: `ASSUMED_DETERMINISTIC`
  - message: `Driver 'inwoners-totaal' has no declared CI; treating as deterministic (zero uncertainty)`
- Confidence band reflects only uncertainty from drivers that declare CI

### REQ-008-004: Low sample count warning
**GIVEN** Monte Carlo simulation constrained to N < 5,000 samples due to time limit

**WHEN** ForecastRun completes

**THEN**:
- ForecastRun.confidence.warnings includes:
  - code: `LOW_SAMPLE_COUNT`
  - message: `Monte Carlo ran with 2,500 samples (target 5,000); CI bands may be underestimated`
- ForecastRun.outputs.confidenceInterval.methodology = 'monte-carlo-2500-samples'

---

## REQ-009: Integration into Begroting-Werkbestand

**Requirement:** The system SHALL allow a ForecastRun (specific scenario, specific horizon) to be promoted into the begroting-werkbestand of `pc-cyclus-workflow` as the basis for that year's programmabegroting; the promotion creates a frozen snapshot.

### REQ-009-001: Promotion to begroting-werkbestand
**GIVEN**:
- ForecastRun for scenario `realistisch-2027`, horizon='4 years', covering 2027-2030
- Status = 'completed'
- Concerncontroller reviews outputs and approves

**WHEN** Concerncontroller clicks "Promote to Begroting 2027"

**THEN**:
- System creates frozen snapshot:
  - Captures state of all referenced Drivers, CostRevenueFormulas, Calibrations, Scenarios at time of ForecastRun
  - Stores JSON copy in ForecastRun.inputSnapshots
- pc-cyclus-workflow receives webhook notification with snapshot
- Begroting-werkbestand 2027 is populated with ForecastRun.outputs (per target, per period)
- ForecastRun.promotedToBegrotingRef = reference to begroting object
- Snapshot is immutable; any future changes to drivers/formulas do not affect begroting

### REQ-009-002: Frozen snapshot with rolling forecast divergence
**GIVEN**:
- Promoted ForecastRun from 2027-04-05 (realistisch-2027, 18 months, covering 2027-04–2028-09)
- Next rolling ForecastRun from 2027-05-05 (same scenario, 18 months, covering 2027-05–2028-10)
- On 2027-06-01, upstream driver 'inwoners-totaal' refreshes with new data

**WHEN** begroting-werkbestand consumes promoted ForecastRun, AND rolling forecast executes 2027-06-05

**THEN**:
- Begroting-werkbestand snapshot remains unchanged (frozen at 2027-04-05 state)
- Rolling-forecast May and June runs use updated driver data
- Dashboard shows divergence: "Begroting 2027 based on 2027-04-05 forecast; current rolling forecast differs by [amount]"
- Divergence attributed to driver updates (via comparison of input snapshots)

### REQ-009-003: Raads-amendement as variance override
**GIVEN**:
- Begroting-werkbestand 2027 promoted from realistisch-2027
- Raad amends Wmo programma budget: +EUR 500,000 (original forecast EUR 12,500,000 → amended EUR 13,000,000)

**WHEN** raads-amendement is recorded in pc-cyclus-workflow

**THEN**:
- Amendment is captured in a related Scenario override (raads-amendement tag)
- Next-period (Q1-2027 close) variance computation includes raads-amendement as override
- Variance.attribution includes `raads-amendement: EUR 500,000` as one component
- Variance commentary: "EUR 500k of delta due to raads-amendement; remaining EUR 200k to be explained"

---

## REQ-010: Sector Reference Packs

**Requirement:** The system SHALL ship reference packs (sets of Drivers + reference formulas + calibration starting points) per sector: gemeente, provincie, waterschap, and per dominante domeinen: sociaal domein, fysieke leefomgeving, jeugd, Wmo, onderwijs, wegen, riolering.

### REQ-010-001: Reference pack seeding
**GIVEN** a fresh financeq installation for Gemeente Zeist

**WHEN** admin runs seed command: `financeq:seed-data driver-based-forecasting:reference-gemeente`

**THEN**:
- System imports reference pack: gemeente sector packs (Wmo, Jeugd, Onderwijs, OZB, Riolering)
- Drivers loaded: inwoners-totaal, leerlingen-primair, leerlingen-voortgezet, wmo-clienten-zwaar, etc. (20–30 drivers per sector)
- CostRevenueFormulas loaded: reference formula templates for each driver domain (5–10 formulas per sector)
- Calibrations loaded: Q4-2024 calibration starting points from NBA/Vektis benchmarks
- All objects created via ObjectService.saveObject() with slug-based deduplication (idempotent)
- AdminUI shows success message: "Imported 28 drivers, 12 formulas, 15 calibrations for gemeente sector"

### REQ-010-002: Reference pack updates with diff
**GIVEN**:
- Reference pack v2026.1 already imported (installed seed date 2026-01-15)
- Reference pack v2027.1 published (released 2027-01-10)

**WHEN** admin runs: `financeq:update-reference-packs driver-based-forecasting:gemeente`

**THEN**:
- System compares old (v2026.1) and new (v2027.1) pack manifests
- Diffs detected:
  - New drivers: `wmo-clienten-licht` (added), 3 others
  - Renamed drivers: `leerlingen-voortgezet-basis` → `leerlingen-voortgezet`
  - Formula changes: Wmo-zorg-zwaar formula updated to include new cost component
  - Calibration updates: 2024–2025 data incorporated, constants recalibrated
- UI displays structured diff with checkboxes:
  ```
  [ ] Add new driver: wmo-clienten-licht
  [ ] Rename driver: leerlingen-voortgezet-basis → leerlingen-voortgezet
  [ ] Update formula: wmo-zorg-zwaar (details show before/after DSL)
  [ ] Update calibration: 5 constants recalibrated
  ```
- Admin reviews and clicks "Apply Updates"
- System applies changes with AuditTrail logging (who, when, what changed)

### REQ-010-003: Waterschap pack filtering
**GIVEN** a fresh financeq installation for Waterschap Zuiderzeeland (waterschap sector)

**WHEN** admin is offered reference packs during setup

**THEN**:
- System filters pack list: gemeente packs de-emphasized or hidden (not applicable)
- Waterschap packs surfaced: "Waterschap Reference Pack 2027"
- Riolering/watertransport domain packs offered (relevant to waterschap operations)
- Admin selects waterschap pack(s) and imports
- Gemeente-specific drivers (e.g., inwoners-totaal for OZB) are NOT imported (irrelevant)

### REQ-010-004: Provincie multi-domain pack
**GIVEN** Provincie Utrecht (provincie sector)

**WHEN** admin seeds with: `financeq:seed-data driver-based-forecasting:reference-provincie`

**THEN**:
- Provincie pack imported (aggregated/higher-level drivers than gemeente)
- Provinces may use gemeente-level granularity or roll-up (both supported)
- Domain packs selected: Wmo, Jeugd (provincies co-finance), Onderwijs (DUO-funded, but provincie contributes to some programs)
- Pack reflects provincie-level cost responsibility (not full municipal cost centers)
